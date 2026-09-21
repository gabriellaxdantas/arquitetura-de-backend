# Observações — Oficina de Ferramentas: Docker, kind e Kubernetes locais

## O que já existia

O laboratório `plataforma-hospitalar` já trazia, do import base do curso, exatamente
o que a oficina do Módulo 6 pede:

- `Dockerfile`: parte de `python:3.12-slim`, instala o pacote com
  `pip install .`, cria o usuário sem privilégios `app` (uid 10001) e expõe a
  porta 8000;
- `infra/k8s/namespace.yaml`, `configmap.yaml`, `deployment.yaml`,
  `service.yaml`, `hpa.yaml`: os cinco manifests com o Deployment de duas
  réplicas, `RollingUpdate` com `maxUnavailable: 0`/`maxSurge: 1`, probes de
  `readiness` (`/health/ready`) e `liveness` (`/health/live`), requests/limits
  de CPU e memória, e HPA de 2 a 5 réplicas;
- `infra/kind/cluster.yaml`: cluster `hospital-local` de um nó
  control-plane, mapeando `30080` do container apenas para
  `127.0.0.1:18080`;
- `src/hospital/api/main.py`: os dois endpoints de saúde usados pelas probes,
  já implementados sem dependência de banco ou serviço externo;
- `tests/test_k8s_manifests.py`: teste que carrega os YAMLs com PyYAML e
  confere estrutura, réplicas, probes, portas, resources e o mapeamento de
  porta do kind — sem precisar de um cluster real.

Nada precisou ser criado ou corrigido no código para esta entrega; o trabalho
desta oficina foi executar o roteiro e documentar o que este ambiente permite
comprovar.

## Validação estática executada (sem Docker/kind/kubectl)

Dentro de `laboratorios/plataforma-hospitalar`, com o `.venv` do laboratório:

```
python -m pytest tests/test_k8s_manifests.py -v
# 3 passed in 0.16s (saída completa em pytest-k8s-manifests.txt)

python -m pytest -q
# 20 passed, 3 skipped in 5.87s (saída completa em pytest-suite-completa.txt)
# os 3 "skipped" dependem de Docker Compose ativo (COMPOSE_LIVE), não de Kubernetes
```

`test_k8s_manifests.py` confirma, lendo o YAML diretamente (sem `kubectl
apply --dry-run=client`, que também depende do binário `kubectl`):

- o Namespace é exatamente `{name: hospital, labels: {app: hospital-api}}`;
- o Deployment está no namespace `hospital`, pede 2 réplicas, estratégia
  `RollingUpdate`, seletor `app: hospital-api`, container `hospital-api` na
  porta nomeada `http` (8000), `resources.requests` de 100m CPU/128Mi e
  `resources.limits` de 250m CPU/256Mi;
- `readinessProbe` aponta para `/health/ready` e `livenessProbe` para
  `/health/live`, e as duas probes **não são iguais** (`readinessProbe !=
  livenessProbe`) — a oficina depende dessa distinção;
- o Service seleciona `app: hospital-api` e expõe a porta 8000 com
  `nodePort: 30080`;
- o HPA (`autoscaling/v2`) referencia o Deployment `hospital-api` com
  `minReplicas: 2` e `maxReplicas: 5`;
- o cluster kind mapeia `containerPort: 30080` para
  `hostPort: 18080` apenas em `listenAddress: 127.0.0.1`;
- o comando do container é exatamente
  `python -m uvicorn hospital.api.main:app --host 0.0.0.0 --port 8000`.

Além do teste automatizado, executei a aplicação FastAPI diretamente com
`uvicorn` **sem container e sem Kubernetes**, só para confirmar que os dois
endpoints usados pelas probes respondem com o contrato exato citado no
roteiro:

```
python -m uvicorn hospital.api.main:app --host 127.0.0.1 --port 8010
curl http://127.0.0.1:8010/health/ready   # 200 {"status":"ready"}
curl http://127.0.0.1:8010/health/live    # 200 {"status":"live"}
```

(saída em `health-ready.txt` e `health-live.txt`). Isso comprova a lógica dos
endpoints, mas **não** comprova nada sobre o Service, o roteamento por labels,
o rollout ou a reconciliação — só o Kubernetes real (ou o kind) demonstraria
isso.

## Limitação deste ambiente: Docker, kind e kubectl indisponíveis

Esta máquina não tem Docker Engine/Desktop instalado (`docker` não é
reconhecido no PATH nem em Git Bash nem em PowerShell; não há pasta
`Program Files\Docker`), e por consequência não há como instalar/usar `kind`
(que depende do daemon Docker) nem foi instalado `kubectl` separadamente.
Por isso **não foi possível**:

- `docker build -t hospital-api:1.0.0 .` e `docker image inspect
  hospital-api:1.0.0`;
- `kind create cluster --name hospital-local --config infra/kind/cluster.yaml`
  e `kind load docker-image hospital-api:1.0.0 --name hospital-local`;
- `kubectl config current-context` confirmando `kind-hospital-local`, nem
  `kubectl apply --dry-run=client` (esse dry-run é do lado do cliente
  `kubectl`, não do YAML puro — por isso a validação real usada aqui foi o
  teste Python, que não depende do binário);
- aplicar os cinco manifests, observar `kubectl rollout status`, `kubectl get
  deployment,pods,service,hpa -n hospital -o wide` e o `EndpointSlice`;
- a demonstração de reconciliação (apagar um Pod e ver o ReplicaSet recriar);
- o cenário de falha controlada: `kubectl set image` com uma tag inexistente,
  observar `ImagePullBackOff`/`ErrImagePull`, confirmar que `maxUnavailable: 0`
  manteve as duas réplicas antigas disponíveis, e então `kubectl rollout undo`
  seguido de `curl` em `/health/live` seguindo pela porta `18080` do host;
- `kind delete cluster --name hospital-local` ao final.

Para gerar essa evidência completa, os comandos abaixo devem ser executados em
uma máquina com Docker Desktop, `kind` e `kubectl` instalados, dentro de
`laboratorios/plataforma-hospitalar` (idênticos aos do roteiro da oficina):

```bash
docker build -t hospital-api:1.0.0 .
kind create cluster --name hospital-local --config infra/kind/cluster.yaml
kind load docker-image hospital-api:1.0.0 --name hospital-local
kubectl config current-context
# esperado: kind-hospital-local

kubectl apply -f infra/k8s/namespace.yaml
kubectl apply -f infra/k8s/configmap.yaml -f infra/k8s/deployment.yaml \
  -f infra/k8s/service.yaml -f infra/k8s/hpa.yaml
kubectl rollout status deployment/hospital-api -n hospital
kubectl get deployment,pods,service,hpa -n hospital -o wide
curl --fail --silent http://127.0.0.1:18080/health/ready
kubectl get endpointslice -n hospital -l kubernetes.io/service-name=hospital-api

# reconciliação
kubectl delete pod -n hospital -l app=hospital-api --field-selector=status.phase=Running -o name | head -n1 | xargs kubectl delete -n hospital
kubectl get pods -n hospital -w   # observar recriação automática

# falha controlada + rollback
kubectl set image deployment/hospital-api hospital-api=hospital-api:imagem-propositalmente-ausente -n hospital
kubectl rollout status deployment/hospital-api -n hospital --timeout=20s || true
kubectl get pods -n hospital
kubectl describe deployment/hospital-api -n hospital
kubectl rollout undo deployment/hospital-api -n hospital
kubectl rollout status deployment/hospital-api -n hospital
curl --fail --silent http://127.0.0.1:18080/health/live

kind delete cluster --name hospital-local
```

## Interpretação

**Capacidade demonstrada nesta entrega:** a validação estática prova, por
inspeção reproduzível (teste automatizado, não leitura manual do YAML), que o
desenho declarado é seguro *antes* de qualquer aplicação: a estratégia de
rollout garante zero réplicas indisponíveis durante atualização
(`maxUnavailable: 0`) permitindo uma réplica extra temporária
(`maxSurge: 1`), e as duas probes são deliberadamente diferentes — a
`readiness` pode remover um Pod do Service sem reiniciá-lo, enquanto a
`liveness` é o único gatilho de reinício do processo. Essa separação de
responsabilidades é o ponto central do módulo e pôde ser comprovada sem
depender de Docker/kind estarem instalados.

**Limite de produção que este laboratório não prova:** nada aqui valida a
cadeia de confiança da imagem. `kind load docker-image` injeta a imagem
construída localmente direto no nó do cluster, sem passar por um registry,
sem autenticação, sem scan de vulnerabilidades e sem qualquer garantia de que
a mesma imagem testada é a que roda em produção. Um ambiente real precisaria
de um registry versionado (ex.: tags imutáveis por SHA de commit), verificação
de proveniência/assinatura de imagem e uma pipeline de CI que impeça publicar
uma tag inexistente — exatamente o cenário de falha que a oficina reproduz de
propósito com `imagem-propositalmente-ausente`.

## Respostas às questões exploratórias

**Que risco existe em executar `kubectl apply` no contexto errado?**
`kubectl` aplica contra o cluster do contexto atual, não contra o cluster que
a pessoa imagina. Se o contexto ativo for um cluster compartilhado (staging,
produção, ou o cluster de um colega), os manifests da oficina — que usam nomes
fixos como `hospital` e `hospital-api` — podem sobrescrever ou remover
recursos reais de outra equipe. Por isso o roteiro insiste em confirmar
`kind-hospital-local` antes de qualquer `apply`.

**Por que o laboratório fixa o acesso em `127.0.0.1`?**
Porque o cluster kind é descartável e local: expor o NodePort em todas as
interfaces (`0.0.0.0`) tornaria a API acessível por qualquer máquina na mesma
rede, sem autenticação, mesmo sendo apenas um exercício. Fixar
`listenAddress: 127.0.0.1` garante que só o próprio computador da aula alcança
a porta 18080.

**Por que `IfNotPresent` faz sentido para a imagem carregada no kind?**
`kind load docker-image` copia a imagem do Docker local para o containerd do
nó; não existe um registry remoto para o kubelet consultar. Com
`imagePullPolicy: IfNotPresent`, o kubelet usa a imagem já carregada no nó em
vez de tentar buscá-la em um registry (que falharia, pois `hospital-api:1.0.0`
não existe em nenhum registry público ou privado).

**Que evidência adicional uma CI produziria para uma imagem de produção?**
Um pipeline de CI produziria: build reprodutível a partir de um commit
específico, tag imutável (por SHA ou versão semântica assinada), resultado de
scan de vulnerabilidades, push autenticado para um registry privado, e
registro de qual manifesto/tag foi de fato implantado — nada disso existe
quando a imagem só é `docker build` local e `kind load`.

**Qual falha seria detectada por readiness mas não deveria reiniciar um
processo?**
Uma dependência externa temporariamente indisponível (por exemplo, uma
migração de banco ainda em andamento ou um serviço downstream fora do ar).
O processo da API continua vivo e saudável — não travou —, mas não deve
receber tráfego enquanto essa dependência não responde. Reiniciar o container
nesse caso não resolveria nada (a dependência externa continuaria fora), por
isso essa checagem pertence à `readiness`, não à `liveness`.

**Por que duas réplicas no kind não equivalem a duas zonas?**
As duas réplicas do Deployment rodam como Pods no mesmo (único) nó
control-plane do kind, que por sua vez é um único container Docker na mesma
máquina física. Zonas de disponibilidade real implicam falha independente de
energia, rede e hardware; no kind, se a máquina do laboratório desligar ou o
daemon Docker cair, as "duas réplicas" caem juntas — não há isolamento de
falha real entre elas.

**Que sinal complementar mostraria que o endpoint ainda atende com uma
réplica fora?**
Repetir a mesma consulta de `readiness`/tráfego várias vezes seguidas durante
a interrupção de uma réplica e observar que todas as respostas continuam
`200`, junto com `kubectl get endpointslice` mostrando que o endereço do Pod
removido já não está na lista de prontos — a ausência do endereço na
EndpointSlice é o sinal de que o Service parou de rotear para aquele Pod antes
mesmo de qualquer requisição falhar.

**Como requests de CPU participam do cálculo de utilização do HPA?**
O HPA de CPU (`autoscaling/v2` do tipo `Resource`) calcula a utilização como a
razão entre o uso corrente medido pelo Metrics Server e o valor de
`resources.requests.cpu` declarado no container (aqui, 100m) — não o `limits`.
Por isso o `request` não é só uma dica de agendamento: ele é o denominador que
decide, junto com o alvo de utilização configurado, quando o HPA aumenta o
número de réplicas entre `minReplicas: 2` e `maxReplicas: 5`.

**Que parte de uma migração de banco `rollout undo` não desfaria?**
`rollout undo` só reverte a especificação do Pod (imagem, variáveis de
ambiente, comando) para uma revisão anterior do Deployment; ele não reverte
mudanças de estado em sistemas externos, como uma migração de esquema de banco
de dados já aplicada. Se a versão nova tivesse alterado uma tabela e a antiga
não souber ler o novo formato, o rollback do Deployment não desfaz a migração
— seria necessária uma migração reversível própria ou uma estratégia de
compatibilidade entre versões.

**Qual política de CI evitaria chegar a uma tag inexistente?**
Uma pipeline que só permite `kubectl set image`/atualização de manifesto
depois que o build correspondente terminou com sucesso e a imagem foi
efetivamente publicada no registry (gate de "build antes de deploy"), aliada a
uma etapa que verifica a existência da tag no registry antes de aplicar o
manifesto — impedindo que alguém aponte o Deployment para uma tag que nunca
chegou a ser construída ou publicada.

**Que carga sintética respeitaria a capacidade da máquina do grupo?**
Uma carga gerada localmente com uma ferramenta simples (por exemplo poucas
dezenas de requisições por segundo contra `/health/ready` ou um endpoint real,
usando um script Python com `httpx` ou uma ferramenta como `hey`/`bombardier`
com limite de taxa explícito), executada por um tempo curto e com o número de
requisições concorrentes limitado ao que o laptop do grupo aguenta sem travar
o próprio Docker Desktop — nunca uma carga "até falhar" sem limite definido
antecipadamente.

**Que métrica além de CPU indicaria uma fila crescente?**
O tamanho da fila de requisições pendentes (ou de uma fila de mensagens
real, como profundidade de fila do RabbitMQ do Módulo 5), a latência de
resposta (p95/p99) crescendo, ou o número de conexões em espera — qualquer
uma dessas cresce quando a demanda ultrapassa a capacidade de processamento
mesmo que a CPU média ainda não esteja saturada, e um HPA baseado só em CPU
não reagiria a tempo a esse tipo de gargalo.
