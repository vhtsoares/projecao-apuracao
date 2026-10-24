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

## Regras da projeção

- A página lê o 2º turno (eleição `6258`) a cada 60 s.
- UFs sem apuração ou com menos de 5% do eleitorado apurado contam como não
  apuradas: entram na projeção com o resultado do 1º turno na UF (eleição
  `6257`), dividido entre os dois finalistas.
