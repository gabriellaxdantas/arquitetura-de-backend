# Notas — Oficina de Ferramentas (Módulo 2: APIs)

## Ambiente
- Python 3.12.10, venv em `.venv/`
- Dependências instaladas com `pip install -e ".[dev]"` (foi necessário instalar também
  `pyyaml`, que os testes importam mas não está listado no `pyproject.toml`)
- Node v24.15.0 / npx 11.12.1
- Spectral CLI 6.16.1 (via `npx`)
- **Bruno não foi utilizado** (ferramenta gráfica não automatizável neste ambiente) — os
  testes manuais de requisição foram feitos via `curl`, produzindo evidência equivalente
  em `evidencias/requisicoes.txt`.
- Extensão opcional do Gateway Ocelot (.NET) **não foi realizada** (fora do escopo desta
  entrega, por decisão explícita).

## Trilha essencial executada
1. `pytest tests -q` → 2 falhas em `test_event_idempotency.py`, não relacionadas ao
   contrato (erro de `PermissionError: WinError 32` ao limpar um arquivo temporário
   SQLite no Windows — problema de ambiente, não do contrato da API).
2. `pytest tests/test_api_contract.py -q` → **7 testes aprovados** (a doc da oficina cita
   6; o repositório evoluiu com mais um teste de contrato).
3. API iniciada com `uvicorn hospital.api.main:app` em `http://127.0.0.1:8000`.
4. `POST /elegibilidades` com corpo válido → `202 Accepted`, cabeçalho `Location` e
   `situacao: recebida` (evidência em `requisicoes.txt`).
5. `GET /elegibilidades/{protocolo}` com o protocolo retornado → `200 OK` com o mesmo
   protocolo.
6. `POST /elegibilidades` sem o campo `cpf` → `422 Unprocessable Entity`, com
   `codigo: dados_invalidos`.
7. `npx spectral lint contratos/openapi.yaml` → "No results with a severity of 'error'
   found!", código de saída 0 (`spectral-valido.txt`).
8. Falha deliberada: copiado o contrato para `openapi-experimento.yaml` e alterado apenas
   o `cpf` do exemplo `pedidoValido` de `'12345678901'` para `'123'`. O Spectral detectou
   o erro `oas3-valid-media-example` ("cpf" property must match pattern "^\d{11}$"), com
   código de saída 1 (`spectral-falha-deliberada.txt`).

## Exploração em dupla (feita sozinha, papéis alternados)
**Papel "só lê o OpenAPI":** o contrato promete, para `POST /elegibilidades`, um `202`
com `Location` e um `protocolo`; para corpo inválido, promete `422` com
`codigo: dados_invalidos`. Também define, via `pattern`, que `cpf` deve ter exatamente
11 dígitos — mas essa validação de padrão só é *anunciada* no schema; nada no contrato
garante que a implementação realmente a aplica.

**Papel "só lê os testes":** `test_api_contract.py` comprova, via `TestClient`, que a API
realmente responde `202`/`200`/`422` nos casos exercitados e que os campos obrigatórios
declarados no schema são de fato exigidos em tempo de execução.

**Promessa documentada e não verificada por teste:** o contrato declara múltiplos
possíveis valores de erro (`codigo`) no schema de erro, mas os testes de contrato
exercitados só cobrem o caminho de `dados_invalidos` — nenhum teste automatizado força,
por exemplo, um erro de protocolo inexistente (`404`) a produzir um `codigo` específico
do schema de erro.

**Asserção que depende do contrato:** o teste que verifica o cabeçalho `Location`
depende diretamente da declaração desse cabeçalho como `required: true` no contrato —
sem essa declaração, o teste ainda passaria, mas deixaria de estar ancorado numa garantia
documentada para o consumidor.

## Extensão — leitura do `.spectral.yaml`
O arquivo define regras sobre: presença de `tags` em cada operação (ajuda ferramentas de
cliente e portais de documentação a agrupar endpoints), presença de `description` em
operações e no documento (ajuda humanos a entender a intenção sem ler o código) e
presença de `operationId` único (permite geração estável de SDKs/clients a partir do
contrato). Uma regra que o linter **não conseguiria decidir sozinho**: se a
`description` de uma operação está *semanticamente correta* em relação ao que o endpoint
de fato faz — o Spectral só verifica que o campo existe e não está vazio, não que o
texto é verdadeiro.

## Questões exploratórias
1. **O que `202` permite ao provedor mudar sem quebrar o consumidor?** Permite que o
   processamento da elegibilidade seja assíncrono e mude de duração ou de implementação
   (síncrono hoje, fila/mensageria depois) sem afetar o contrato do consumidor, que já
   está preparado para "aceito, consulte depois" em vez de esperar o resultado final na
   mesma chamada.
2. **Por que `Location` é melhor que pedir ao consumidor para montar a URL por
   convenção?** Porque o formato da URL do recurso deixa de ser um conhecimento
   implícito compartilhado (e, portanto, uma fonte de acoplamento e de erros se o
   formato mudar); o servidor comunica explicitamente onde o recurso pode ser consultado.
3. **Qual divergência entre OpenAPI e aplicação os testes atuais ainda não detectam?**
   Os testes não comparam automaticamente o schema de erro *documentado* com o
   `app.openapi()` *gerado* pela aplicação em execução — poderia haver um código de erro
   novo implementado no código sem que o contrato declarado o descreva, sem que nenhum
   teste falhe.
4. **Quando uma chave de idempotência passaria a ser necessária?** Quando o cliente
   puder reenviar o mesmo `POST /elegibilidades` (por timeout, retry de rede etc.) e o
   sistema precisar garantir que isso não gere dois protocolos/efeitos de negócio
   distintos para o mesmo pedido lógico.
5. **Que parte do experimento deixaria de funcionar com duas instâncias e memória
   separada?** O armazenamento dos protocolos criados por `POST` estar em memória do
   processo Uvicorn: um `GET /elegibilidades/{protocolo}` roteado para uma segunda
   instância (sem estado compartilhado) retornaria `404`, mesmo que o protocolo tenha
   sido criado com sucesso na primeira instância.
