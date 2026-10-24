# Plano — Acompanhamento e projeção da apuração (2º turno, Presidente, 2026)

## 1. Objetivo

Criar uma página que lê, em tempo real, a apuração do 2º turno presidencial em
cada UF (e no exterior) e projeta o resultado nacional supondo que, em cada UF,
os percentuais já apurados se mantêm até 100%.

## 2. Fonte de dados (verificada em 24/10/2026)

O TSE não publica uma "API" documentada, mas o site oficial de resultados
(`resultados.tse.jus.br`) lê arquivos JSON estáticos que podem ser consumidos
diretamente.

| Item | Valor |
|---|---|
| Configuração geral | `https://resultados.tse.jus.br/oficial/comum/config/ele-c.json` |
| Código da eleição, 1º turno (federal) | `6257` |
| Código da eleição, 2º turno (federal) | `6258` (campo `cdt2`) |
| Cargo Presidente | `c0001` |
| Arquivo por abrangência | `https://resultados.tse.jus.br/oficial/ele2026/{eleicao}/dados/{uf}/{uf}-c0001-e00{eleicao}-u.json` |
| Abrangências | `br` (nacional), 27 UFs (`ac` … `to`) e `zz` (exterior) |

Exemplos:

- Nacional, 2º turno: `.../ele2026/6258/dados/br/br-c0001-e006258-u.json`
  (já publicado, zerado: `pst = 0,00`, `ts = 499248` seções)
- SP, 1º turno: `.../ele2026/6257/dados/sp/sp-c0001-e006257-u.json`

Campos usados (todos chegam como **string**; percentuais usam vírgula decimal):

| Campo | Significado |
|---|---|
| `dg`, `hg` | data/hora da última atualização do arquivo |
| `s.ts`, `s.st`, `s.pst` | seções totais, seções totalizadas, % de seções totalizadas |
| `e.te`, `e.est` | eleitorado total, eleitorado das seções totalizadas |
| `e.c`, `e.a` | comparecimento, abstenção (nas seções totalizadas) |
| `v.vv`, `v.vb`, `v.tvn` | votos válidos, brancos, nulos |
| `carg[0].agr[].par[].cand[]` | candidatos: `n` (número), `nmu` (nome de urna), `vap` (votos), `pvap` (% válidos) |

Comportamento do servidor:

- **CORS:** o servidor devolve `Access-Control-Allow-Origin` com a origem que fez
  a requisição, logo uma página estática pode buscar os arquivos direto do
  navegador, sem backend nem proxy.
- **Cache:** `Cache-Control: max-age` entre 40 e 60 s, com `ETag` e
  `Last-Modified`. Não adianta consultar com mais frequência que isso.
- **Limite de taxa:** durante a investigação, rajadas de requisições receberam
  `403 Access Denied` (Akamai). Escalone as requisições e evite polling agressivo.

## 3. Metodologia de projeção

Para cada abrangência `u` (27 UFs + `zz`):

1. Calcule a fração apurada pelo **eleitorado**, e não pelas seções (as seções
   têm tamanhos diferentes):
   `f_u = e.est / e.te`
2. Projete os votos de cada candidato `k`:
   `V̂_{u,k} = vap_{u,k} / f_u`
   (equivale a supor que o comparecimento e a divisão dos votos nas seções
   restantes da UF são iguais aos das seções já apuradas).
3. Some as projeções de todas as UFs:
   `V̂_k = Σ_u V̂_{u,k}`, e `%̂_k = V̂_k / Σ_j V̂_j` (percentual de votos válidos).
4. Exiba, lado a lado, o **resultado parcial oficial** (arquivo `br`) e a
   **projeção**, além da diferença projetada em votos e em pontos percentuais.

Tratamento de casos especiais:

- **UF com 0% apurado:** não há dados para projetar. Use, nesse caso, um
  *fallback* configurável:
  - (padrão) resultado do 1º turno na UF, normalizado entre os dois finalistas,
    aplicado ao comparecimento do 1º turno; ou
  - excluir a UF e sinalizar "sem dados" na tabela.
  A UF que estiver usando *fallback* deve aparecer destacada.
- **Apuração muito baixa (< ~5%):** marque a projeção da UF como "instável".
- **Exterior (`zz`):** trate como mais uma abrangência; ela costuma ser apurada
  por último.
- Nunca use `vap` como número de ponto flutuante a partir de strings com vírgula:
  converta com `parseInt` (votos) e substitua `,` por `.` nos percentuais.

Limitação conhecida (deve aparecer na página): a ordem da apuração dentro de uma
UF não é aleatória — capitais e municípios grandes costumam totalizar em ritmos
diferentes do interior. A projeção por UF corrige o viés **entre** estados, mas
não o viés **dentro** de cada estado. Um refinamento possível (fase 2) é projetar
por município, se os arquivos municipais estiverem acessíveis.

## 4. Arquitetura

Uma única página HTML estática, sem backend, publicada como Artifact:

```
index.html
 ├─ config: ELEICAO=6258, ELEICAO_T1=6257, CARGO=c0001, UFs[28], INTERVALO=60s
 ├─ fetchUF(uf)       → busca e normaliza o JSON de uma abrangência
 ├─ cicloAtualizacao  → busca as 28 abrangências escalonadas (~2 s entre elas)
 │                      + o arquivo `br`; ignora respostas sem mudança (dg/hg)
 ├─ projetar(dados)   → aplica a metodologia da seção 3
 └─ render            → painel, tabela por UF, mapa/gráfico
```

- Busque os dados do 1º turno (6257) **uma vez** na carga da página, apenas para
  o *fallback*.
- Mantenha o último resultado válido de cada UF em memória; se uma requisição
  falhar (403, timeout), reutilize o dado anterior e mostre o horário dele.
- Guarde uma série histórica (horário × % apurado × projeção) em memória para
  o gráfico de evolução. Opcional: salvar no `localStorage` para sobreviver a
  um recarregamento.

## 5. Interface

1. **Cabeçalho:** % de seções apuradas no país, horário da última atualização do
   TSE e do último ciclo, indicador de "ao vivo"/erro.
2. **Placar:** para cada candidato, % parcial oficial × % projetado, votos
   projetados e diferença projetada.
3. **Gráfico de evolução:** linha da projeção (%) ao longo da noite, com o %
   parcial oficial como linha de referência. Isso mostra se a projeção converge.
4. **Tabela por UF:** % apurado, % de cada candidato, votos projetados, saldo
   projetado do candidato líder, marcação de *fallback*/instável. Ordenável.
5. **Mapa ou grade de UFs (opcional):** cor pelo líder, intensidade pela margem.
6. **Nota metodológica** curta e o aviso de que o resultado oficial é o do TSE.

Requisitos: funcionar em celular (a maioria vai acompanhar pelo telefone), tema
claro/escuro, números formatados em pt-BR.

## 6. Etapas de implementação

1. **Coleta:** implemente `fetchUF` e o ciclo escalonado; teste com o 1º turno
   (6257), que tem dados completos.
2. **Projeção:** implemente `projetar` e valide com o 1º turno:
   - com dados a 100%, a projeção deve igualar o resultado oficial;
   - simule apuração parcial (ex.: reduzir `est` e os votos proporcionalmente) e
     verifique se a projeção se recupera.
3. **Interface:** placar, tabela e gráfico.
4. **Robustez:** tratamento de 403/timeout, *backoff* exponencial, indicador de
   dados desatualizados.
5. **Modo simulação:** um seletor "1º turno (teste) / 2º turno (ao vivo)" para
   demonstrar a página antes de a apuração começar.
6. **Publicação** como Artifact e teste em celular.

## 7. Cronograma

O 2º turno ocorre em **25/10/2026**; a divulgação começa às 17h (Brasília),
com o fechamento das urnas em todo o país. As etapas 1 a 3 e 5 precisam estar
prontas antes disso; a etapa 4 é desejável.

## 8. Riscos

| Risco | Mitigação |
|---|---|
| Bloqueio por excesso de requisições (403 Akamai) | intervalo ≥ 60 s, requisições escalonadas, *backoff* |
| Mudança de formato/caminho dos arquivos | reler `ele-c.json` na carga e validar o esquema; mensagem clara de erro |
| Viés da ordem de apuração dentro da UF | aviso na página; projeção por município como fase 2 |
| Projeção instável no início da apuração | marcar UFs com pouca apuração; mostrar faixa/aviso até ~10% nacional |
