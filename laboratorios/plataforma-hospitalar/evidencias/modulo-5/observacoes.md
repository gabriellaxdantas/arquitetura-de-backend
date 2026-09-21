# Observações — Oficina de Ferramentas: RabbitMQ e Consumidor Idempotente

## O que já existia e o que foi corrigido

O código de `src/hospital/eventos/publicador.py` e `src/hospital/eventos/consumidor.py`,
o `infra/compose.eventos.yml` e `tests/test_event_idempotency.py` já implementavam
exatamente o contrato pedido pela oficina (exchange `hospital.events`, fila
`billing.resultados.v1`, DLX `hospital.events.dlx`, DLQ `billing.resultados.v1.dlq`,
deduplicação por `event_id` em SQLite).

Ao rodar os testes localmente (Windows), os dois testes que não dependem do broker
falhavam com `PermissionError: [WinError 32]` durante a limpeza do `TemporaryDirectory`.
Causa: `ProcessedEventStore` abria a conexão SQLite com
`with self._connect() as connection:` — em Python, o *context manager* de
`sqlite3.Connection` só controla commit/rollback da transação, **não fecha a conexão**.
A conexão ficava aberta e o Windows bloqueia a exclusão de um arquivo com handle aberto
(o Linux é mais tolerante a isso, por isso o problema não aparecia lá). Corrigido
fechando explicitamente a conexão em cada método (`__init__`, `record`, `attempts_for`,
`business_effect_count`) com `try/finally`. Após a correção:

```
2 passed, 1 skipped in 0.30s
```

(saída completa em `testes-idempotencia.txt`)

## Limitação deste ambiente: Docker indisponível

Este ambiente de execução não tem Docker Engine/Desktop instalado (`docker` não é
reconhecido no PATH). Por isso **não foi possível**:

- subir `infra/compose.eventos.yml` (`docker compose up -d --build --wait`);
- publicar/consumir eventos contra um RabbitMQ real e observar
  `processed=True attempts=1` seguido de `processed=False attempts=2` em processos
  reais de linha de comando;
- consultar a API de management (`/api/queues/%2F/billing.resultados.v1.dlq`) para
  confirmar mensagens na DLQ;
- rodar `COMPOSE_LIVE=1 python -m pytest tests/test_event_idempotency.py -q`
  (o terceiro teste, que exercita o broker de verdade, ficou `SKIPPED` — o resultado
  correto quando `COMPOSE_LIVE` não está definido, não um erro).

O que **foi** validado sem o broker (evidência real, não simulada):

- os dois testes que exercitam a lógica pura do consumidor (deduplicação por
  `event_id` e rejeição de schema inválido sem derrubar o consumidor) passam;
- a lógica de `ProcessedEventStore.record` foi inspecionada e confirma o comportamento
  esperado: primeira chamada grava em `processed_events` e `billing_effects`
  (`processed=True, attempts=1`); chamadas seguintes com o mesmo `event_id` só
  incrementam `attempts` em `processed_events` (`processed=False, attempts=N`),
  sem novo `INSERT` em `billing_effects`.

Para gerar a evidência completa (saídas de console reais, DLQ populada), rode em uma
máquina com Docker Desktop/Engine ativo, dentro de
`laboratorios/plataforma-hospitalar`:

```powershell
$env:RABBITMQ_PORT = "15672"
$env:RABBITMQ_MANAGEMENT_PORT = "15673"
$env:RABBITMQ_URL = "amqp://guest:guest@localhost:15672/"

docker compose -f infra/compose.eventos.yml up -d --build --wait
python -m hospital.eventos.consumidor --once --store evidencias/modulo-5/processed-events.sqlite3

python -m hospital.eventos.publicador --event-id 3fa85f64-5717-4562-b3fc-2c963f66afa6
python -m hospital.eventos.consumidor --once --store evidencias/modulo-5/processed-events.sqlite3
# esperado: processed=True attempts=1

python -m hospital.eventos.publicador --event-id 3fa85f64-5717-4562-b3fc-2c963f66afa6
python -m hospital.eventos.consumidor --once --store evidencias/modulo-5/processed-events.sqlite3
# esperado: processed=False attempts=2

python -m hospital.eventos.publicador --event-id 65e95d82-4f8c-4e93-9bb3-3e0e92deaf1d --invalid
python -m hospital.eventos.consumidor --once --store evidencias/modulo-5/processed-events.sqlite3
# mensagem rejeitada -> vai para a DLQ

curl.exe --fail --silent --user guest:guest `
  "http://localhost:15673/api/queues/%2F/billing.resultados.v1.dlq" | ConvertFrom-Json

$env:COMPOSE_LIVE = "1"
python -m pytest tests/test_event_idempotency.py -q
# esperado: 3 passed

docker compose -f infra/compose.eventos.yml down -v
```

## Por que isso demonstra entrega confiável com idempotência, e não *exactly-once*

O RabbitMQ, como a maioria dos brokers baseados em AMQP, garante **at-least-once
delivery**: uma mensagem confirmada (`ack`) só sai da fila depois que o consumidor
processa com sucesso; se o consumidor cair ou não confirmar a tempo, a mensagem é
reentregue. Isso significa que o **mesmo evento pode chegar mais de uma vez** — é
exatamente o que o teste `test_duplicate_event_has_one_business_effect_and_two_attempts`
reproduz publicando duas vezes o mesmo `event_id`.

O broker não elimina duplicatas por conta própria (não há "exactly-once" de verdade em
sistemas distribuídos sem coordenação adicional). Quem garante que o **efeito de
negócio** aconteça uma única vez é o **consumidor**, através de idempotência: ele
consulta `processed_events` por `event_id` antes de aplicar o efeito e só grava em
`billing_effects` na primeira vez. Tentativas repetidas incrementam `attempts` mas não
duplicam o efeito. Ou seja:

- **at-least-once** descreve a garantia do transporte (a mensagem chega, possivelmente
  mais de uma vez);
- **idempotência** é a técnica na borda do consumidor que torna o *efeito observável*
  equivalente a "exactly-once", mesmo sem essa garantia existir no transporte.

## Cenários onde Kafka representaria uma extensão natural

- **Retenção e replay:** o RabbitMQ remove a mensagem da fila assim que é confirmada;
  o Kafka retém o log de eventos por um período configurável (ou indefinidamente),
  permitindo reprocessar o histórico inteiro — útil para reconstruir projeções,
  corrigir bugs em consumidores ou popular um novo serviço.
- **Múltiplos grupos de consumidores independentes:** no RabbitMQ, uma fila é
  consumida uma única vez (mesmo com vários consumidores, eles competem pelas
  mensagens); no Kafka, vários *consumer groups* podem ler o mesmo tópico
  independentemente, cada um na sua própria posição (offset) — por exemplo,
  faturamento e auditoria consumindo `laboratory.result.available.v1` sem que um
  interfira no outro.
- **Ordenação e particionamento em alta escala:** o Kafka particiona por chave
  (ex.: `patient_id`), preservando ordem por partição em um volume muito maior de
  eventos, cenário em que o modelo de fila única do RabbitMQ passa a ser um gargalo.

## Respostas às questões exploratórias

**Em qual etapa uma queda poderia gerar redelivery?**
Entre o consumidor receber a mensagem (`queue.get`/`basic.consume`) e confirmar o
`ack` — se o processo cair, travar ou perder a conexão AMQP nesse intervalo (por
exemplo, logo depois de gravar no SQLite mas antes do `ack` chegar ao broker), o
RabbitMQ reentrega a mensagem para outro consumidor (ou o mesmo, ao reconectar).

**Por que confirmar (`ack`) antes do SQLite seria inseguro?**
Se o `ack` for enviado antes de persistir o efeito, uma queda do processo entre o
`ack` e a escrita no SQLite faz a mensagem ser removida da fila sem que o efeito de
negócio tenha sido de fato gravado — perda silenciosa do evento. Por isso a ordem
correta é: processar e persistir primeiro, confirmar depois (`message.process()` só
dá `ack` se o bloco terminar sem exceção).

**Qual mudança exigiria uma nova versão do evento?**
Qualquer mudança incompatível com os consumidores existentes: remover ou renomear um
campo obrigatório, mudar o tipo/semântica de um campo (ex.: `result_reference` deixar
de ser um caminho e virar uma URL assinada com expiração), ou tornar obrigatório um
campo hoje ausente. Como o modelo usa `extra="forbid"`, adicionar um campo novo
*opcional* não quebra o contrato atual, mas qualquer coisa que mude o significado dos
campos existentes exige `ResultadoLaboratorialDisponibilizadoV1` virar `...V2`, com uma
`routing_key` própria (`laboratory.result.available.v2`) para permitir que produtor e
consumidores migrem de forma independente.

**Como a DLQ muda com novos `event_id`s?**
Cada mensagem rejeitada (schema inválido) é encaminhada individualmente para a DLQ,
mantendo seu payload original. Novos `event_id`s inválidos apenas se acumulam como
novas mensagens na mesma fila `billing.resultados.v1.dlq` — a DLQ não deduplica nem
agrega; ela existe para inspeção manual e reprocessamento posterior, não para lógica
de negócio.

**Por que há entrega pelo menos uma vez com idempotência, e não exactly-once?**
Porque garantir exactly-once de ponta a ponta exigiria coordenação transacional entre
o broker e o efeito de negócio (ex.: um commit atômico único cobrindo "retirar da fila"
e "aplicar o efeito"), o que RabbitMQ não oferece nativamente entre sistemas
heterogêneos (fila + banco de dados separados). A alternativa prática e amplamente
usada é aceitar at-least-once no transporte e mover a garantia de unicidade do efeito
para o consumidor via idempotência — como feito aqui com a tabela `processed_events`.
