# Projeção da Apuração 2026

Página que acompanha a apuração do 2º turno presidencial de 2026 em cada UF e
projeta o resultado nacional supondo que, em cada UF, os percentuais já apurados
se mantêm até 100%. O plano e a metodologia estão em [PLANO.md](PLANO.md).

## Como usar

A página é um único arquivo, `index.html`, sem dependências nem servidor. Ela lê
os arquivos JSON públicos do TSE direto do navegador.

- **No computador:** baixe `index.html` e abra-o no navegador (duplo clique).
- **Online:** ative o GitHub Pages no repositório (Settings → Pages → Deploy from
  a branch, escolhendo o branch e a pasta raiz) e abra o endereço gerado.

## Modos

| Modo | O que faz |
|---|---|
| 2º turno (ao vivo) | lê a eleição `6258` a cada 60 s |
| 1º turno (teste com dados reais) | lê a eleição `6257`, já 100% apurada; a projeção deve igualar o resultado oficial |
| Simulação de apuração (1º turno) | revela os votos do 1º turno aos poucos, com viés dentro de cada UF, para mostrar a projeção convergindo |

A opção "UFs sem apuração" define se uma UF ainda sem urnas apuradas entra na
projeção com o resultado do 1º turno (dividido entre os dois finalistas) ou fica
de fora.
