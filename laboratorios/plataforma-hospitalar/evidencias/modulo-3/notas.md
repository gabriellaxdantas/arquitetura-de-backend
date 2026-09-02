# Notas — Oficina de Ferramentas (Módulo 3: Serviços)

## Ambiente
- Docker Desktop 4.30.0 (engine 26.1.1), Docker Compose v2.27.0-desktop.2 — daemon
  precisou ser iniciado manualmente antes da execução (`Docker Desktop.exe`).
- Python 3.12.10 instalado via `winget` (não havia interpretador real no PATH, apenas o
  alias stub da Microsoft Store); venv criado em `.venv/` e dependências instaladas com
  `pip install -e ".[dev]"`.
- Portas customizadas via `ELEGIBILIDADE_PORT=18001` e `EXAMES_PORT=18002` para evitar
  conflito com outros serviços locais.

## Trilha executada (todas as evidências em `evidencias/modulo-3/`)
1. `docker compose -f infra/compose.servicos.yml config --quiet` e `--services` →
   validação estática confirma os quatro serviços declarados: `db_elegibilidade`,
   `db_exames`, `elegibilidade`, `exames` (`compose-config-quiet.txt`,
   `compose-config-services.txt`).
2. `docker compose ... up -d --build --wait` → build das quatro imagens e subida com
   sucesso; todos os contêineres reportaram `healthy` (`compose-up.txt`, `compose-ps.txt`).
3. `GET /health` em `elegibilidade` e `exames` → ambos `200 OK`
   (`health-elegibilidade.txt`, `health-exames.txt`).
4. `POST /exames` com beneficiário elegível → `201 Created`, corpo com `solicitacao_id`
   e `situacao: solicitado` (`requisicao-sucesso.txt`).
5. `docker compose stop elegibilidade` → serviço de elegibilidade parado
   (`stop-elegibilidade.txt`).
6. Repetição do `POST /exames` com a dependência fora do ar → `503 Service Unavailable`
   com `codigo: dependencia_indisponivel` (`requisicao-falha-parcial.txt`). Ao mesmo
   tempo, `GET /health` do próprio `exames` continuou respondendo `200 OK`
   (`health-exames-durante-falha.txt`) — evidência direta de que a indisponibilidade de
   uma dependência não derruba a capacidade do serviço de reportar seu próprio processo
   como vivo, mesmo incapaz de completar a operação de negócio.
7. `docker compose up -d --wait` (recuperação) e `pytest tests/test_service_boundaries.py
   -q` → **4 testes aprovados**, incluindo
   `test_exames_makes_its_own_database_failure_observable`
   (`compose-recovery.txt`, `pytest-service-boundaries.txt`).
8. `docker compose down -v` → contêineres, volumes e redes removidos por completo
   (`compose-down.txt`, `compose-ps-after-down.txt` mostra lista vazia).

## Observação sobre a suíte completa de testes
`pytest tests -q` (todos os módulos) falha na coleta de `test_api_contract.py` e
`test_k8s_manifests.py` por `ModuleNotFoundError: No module named 'yaml'` — `pyyaml` é
importado pelos testes mas não está listado em `pyproject.toml`. Isso é um problema de
ambiente pré-existente dos módulos 2 e 5, fora do escopo desta oficina (Módulo 3); os
testes relevantes para este módulo (`test_service_boundaries.py`) não dependem de `yaml`
e passaram integralmente (`pytest-suite-completa.txt` documenta o erro para rastreabilidade).

## Observações-chave

**Isolamento de rede:** o `compose.servicos.yml` declara `elegibilidade-db-net` e
`exames-db-net` como redes internas (`internal: true`) e específicas por par
serviço/banco. O serviço `exames` não tem rota de rede para `db_elegibilidade` — mesmo
código que tentasse consultar o banco de elegibilidade diretamente falharia por
inexistência de rota, não apenas por falta de credenciais.

**Distinção entre "vivo" e "capaz":** o `/health` de cada serviço responde a partir do
próprio processo, sem depender de terceiros. Isso ficou evidente no passo 6: `exames`
seguiu "saudável" enquanto sua operação de negócio (`POST /exames`) ficou indisponível
por causa da dependência externa — dois conceitos diferentes de disponibilidade.

**Tradução semântica de erros:** o serviço `exames` traduz falhas da dependência
`elegibilidade` em códigos com significado de negócio distintos: indisponibilidade
técnica → `503 dependencia_indisponivel`; beneficiário desconhecido (`404` na
elegibilidade) → `422 beneficiario_desconhecido`; contrato de resposta inválido →
`502 contrato_invalido`; beneficiário inelegível → `422 beneficiario_inelegivel`. O
cliente de `exames` recebe informação acionável em vez de um genérico "erro".
