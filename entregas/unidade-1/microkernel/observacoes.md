# Observações — Microkernel (Faturamento por Plugins)

## Condição alterada
No plugin `ImpostoSPPlugin` (`plugins/impostos_sp.py`), a alíquota de ICMS para a
categoria `"eletronico"` foi alterada de `0.12` (12%) para `0.15` (15%), dentro do
dicionário `ALIQUOTAS` usado exclusivamente por esse plugin de imposto de São Paulo.

## O que a saída revelou
- **Fatura #1001 (São Paulo, Notebook Dell + Suporte Técnico):** o imposto `ICMS-SP`
  subiu de R$1.300,00 para R$1.600,00, elevando o total da fatura de R$13.400,00 para
  R$13.700,00. A notificação por e-mail (plugin `NotificacaoEmailPlugin`) refletiu o novo
  total automaticamente, sem qualquer alteração no próprio plugin de notificação.
- **Fatura #1003 (São Paulo, Servidor Dell — item de alto valor):** `ICMS-SP` subiu de
  R$9.000,00 para R$11.250,00, elevando o total de R$84.000,00 para R$86.250,00. O frete
  continuou grátis (regra "acima de R$5k"), pois essa regra pertence a outro plugin
  (`FreteCorrespondenciaPlugin`) e não foi tocada.
- **Fatura #1002 (Rio de Janeiro)** e **Fatura #1004 (Minas Gerais)** não mudaram, pois o
  plugin alterado só age quando `fatura.cliente.estado == "SP"`.

## Responsabilidade arquitetural relacionada
O núcleo (`nucleo.py`, `CoreFaturamento`) não conhece alíquotas, estados ou categorias de
produto — ele só conhece o contrato genérico "plugin de imposto processa uma fatura e
devolve um resultado". Toda a regra fiscal específica de SP vive isolada dentro de
`ImpostoSPPlugin`. A mudança de alíquota alterou o valor calculado por um único plugin, e
esse efeito se propagou corretamente para o total e para a notificação **sem exigir
nenhuma alteração no núcleo, nos plugins de RJ/frete/notificação, nem no domínio**. Isso
demonstra a responsabilidade estável do núcleo (orquestrar registro e execução de
plugins) versus a responsabilidade variável de cada plugin (a regra de negócio em si).
