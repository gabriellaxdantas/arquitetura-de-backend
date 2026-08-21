# Observações — Pipes and Filters (Triagem de Currículos)

## Condição alterada
No `main.py`, o requisito `experiencia_minima` da vaga foi alterado de `3` para `4` anos.
Essa constante é lida pelo filtro `FiltroPorExperienciaMinima` (em `filtros/testers.py`),
que descarta (reprova) qualquer currículo com `anos_experiencia` abaixo do mínimo da vaga.

## O que a saída revelou
- Na saída **antes**, apenas Bruno Rocha (1 ano) era reprovado por experiência; Ana Lima,
  Elena Souza, Diego Faria eram aprovados — total de **3 candidatos** aprovados, com Elena
  Souza (3 anos) aparecendo em 2º lugar no ranking com score de 100%.
- Na saída **depois**, além de Bruno Rocha, **Elena Souza também é reprovada** (agora tem
  exatamente o mínimo antigo, mas menos que o novo mínimo de 4), reduzindo o resultado
  final para **2 candidatos aprovados** (Ana Lima e Diego Faria). O ranking mudou: Diego
  Faria passa a ocupar a 2ª posição que antes era de Elena.

## Responsabilidade arquitetural relacionada
O critério de corte é responsabilidade exclusiva de um filtro do tipo **tester**
(`FiltroPorExperienciaMinima`), que apenas aprova ou descarta, sem transformar dados.
A mudança não exigiu tocar no `producer` (leitura), nos `transformers` (normalização e
cálculo de score) nem no `consumer` (relatório/ranking) — apenas no parâmetro de entrada
(`vaga.experiencia_minima`) consumido pelo filtro certo. Isso evidencia que, no estilo
pipes-and-filters, cada filtro tem uma responsabilidade única e o pipeline como um todo
reage à mudança de forma previsível: menos aprovados na entrada de um filtro reduz o que
chega ao `CalculadorDeScore` e ao `RelatorioDeTriagem`, sem que a lógica de ranking em si
precise mudar.
