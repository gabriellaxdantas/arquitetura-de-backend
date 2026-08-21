# Observações — Estilo em Camadas (Agenda Clínica)

## Condição alterada
Adicionado um buffer mínimo de 10 minutos entre consultas do mesmo médico na regra
`Horario.conflita_com` (arquivo `dominio.py`). Antes, dois horários só conflitavam se
houvesse sobreposição estrita de intervalos. Depois, também conflitam se o intervalo
entre o fim de um e o início do outro for menor que 10 minutos.

Também foi adicionado um novo cenário no `main.py` ("2b") que tenta agendar uma consulta
para a Dra. Ana das 10:30 às 11:00, imediatamente após a consulta #2 (10:00–10:30) da
mesma médica — um caso que **não conflitava** pela regra antiga.

## O que a saída revelou
- Na saída **antes**, esse cenário não existia (a consulta às 10:30 seria aceita, pois
  não há sobreposição de intervalos).
- Na saída **depois**, a mesma tentativa retorna `HTTP 409 CONFLICT`, com a mensagem
  "Dr(a). Dra. Ana Silva já tem consulta das 10:00 às 10:30.", porque o novo buffer passa
  a considerar os 10 minutos seguintes ao fim da consulta anterior como indisponíveis.

## Responsabilidade arquitetural relacionada
A regra de negócio (o que conta como conflito de agenda) está isolada na **camada de
domínio** (`Horario.conflita_com`), não na camada de serviço nem na de apresentação.
Isso confirma a separação de responsabilidades do estilo em camadas: foi possível mudar
uma regra de negócio sensível (o que é "conflito") sem tocar em `servicos.py`,
`repositorios.py` ou `apresentacao.py`. A camada de serviço (`AgendamentoServico`)
continua apenas orquestrando a chamada à regra do domínio, e a camada de apresentação
continua apenas traduzindo o resultado em códigos HTTP.
