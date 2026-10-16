# Guia da Pessoa 3 — Kubernetes (YAMLs + Deploy + Self-healing + Scaling)

---

## 1. Objetivo

Escrever todos os arquivos YAML do Kubernetes, fazer o deploy no Minikube e demonstrar:
- **Self-healing:** deletar um Pod e ver o Kubernetes recriar automaticamente.
- **Scaling:** aumentar o número de réplicas automaticamente com o HPA.

---

## 2. O que precisa estar instalado

- WSL2 com Ubuntu
- Docker Desktop
- Minikube
- kubectl
- Git

Se algum desses não estiver instalado, peça ajuda no grupo antes de continuar.

---

## 3. Dependências entre pessoas

Esta guia tem **duas dependências**:

| Passo | Depende de | O que precisa |
|-------|-----------|---------------|
| 5.5 (deploy real) | Pessoa 1 | Imagem `process-api:latest` buildada |
| 5.5 (deploy real) | Pessoa 2 | Imagens `nginx-cron:latest` e `cron-daemon:latest` buildadas |

**Importante:** você **não precisa esperar** para começar. Você pode:
1. Escrever todos os YAMLs (passos 5.1 a 5.4).
2. Testar com **imagens genéricas** (ex: `nginx:latest`) enquanto as reais não ficam prontas.
3. Quando as imagens reais chegarem, é só carregar no Minikube e aplicar os YAMLs de novo.

**Avise no grupo** assim que terminar as demonstrações de self-healing e scaling. A Pessoa 4 precisa dos seus resultados para escrever o artigo.

---

## 4. Teoria essencial

### 4.1. O que é Kubernetes

**Kubernetes** (K8s) é um orquestrador de containers. Ele automatiza:
- Deploy de aplicações containerizadas.
- Escalonamento (aumentar/diminuir réplicas).
- Gerenciamento do ciclo de vida dos containers.
- Self-healing (recriar containers que falham).
- Balanceamento de carga.

**Por que existe?** Docker sozinho resolve 1 container. Kubernetes resolve centenas de containers em vários servidores.

### 4.2. Arquitetura do Kubernetes

Um cluster tem dois tipos de nós:

**Control Plane (cérebro):**
| Componente | Função |
|------------|--------|
| kube-apiserver | Expõe a API REST do Kubernetes |
| etcd | Banco de dados que guarda todo o estado do cluster |
| kube-scheduler | Decide em qual nó cada Pod roda |
| kube-controller-manager | Executa os controladores (ReplicaSet, etc.) |

**Worker Nodes (executam containers):**
| Componente | Função |
|------------|--------|
| kubelet | Agente que garante que os Pods rodem |
| kube-proxy | Gerencia rede dos Services |
| Container Runtime | Executa os containers (containerd) |

**Pergunta provável do professor:** "O que é o etcd?"
**Resposta:** Banco de dados chave-valor distribuído que armazena todo o estado do cluster.

### 4.3. Conceitos fundamentais

**Pod:**
- Menor unidade implantável no Kubernetes.
- Contém 1 ou mais containers que compartilham rede e armazenamento.
- Efêmero: nasce, morre e é substituído.

**Deployment:**
- Gerencia um conjunto de Pods.
- Define quantas réplicas, qual imagem, quais labels.
- Garante self-healing e atualizações graduais.

**ReplicaSet:**
- Controlador que garante o número desejado de réplicas.
- Criado automaticamente pelo Deployment.

**Service:**
- Expõe um conjunto de Pods como serviço de rede.
- Tipos: ClusterIP (interno), NodePort (porta do nó), LoadBalancer (externo).

**Namespace:**
- "Pasta" lógica para organizar recursos no cluster.

**Labels e Selectors:**
- **Labels:** etiquetas nos Pods (ex: `app: process-api`).
- **Selectors:** como o Service/Deployment encontra os Pods.

**HPA (Horizontal Pod Autoscaler):**
- Ajusta automaticamente o número de réplicas com base em métricas (CPU).
- Você define mínimo e máximo de réplicas.

**hostPID:**
- `hostPID: true` faz o container compartilhar o namespace de PID do host.
- Permite que a API veja todos os processos do nó.
- Sem isso, o container só vê seus próprios processos.

### 4.4. Self-healing

O Kubernetes tem um **loop de reconciliação**:
1. Você define o **estado desejado** (ex: 2 réplicas).
2. O controlador observa o **estado atual** (ex: 1 réplica).
3. O controlador age para corrigir (cria nova réplica).
4. Esse loop roda continuamente.

**Quem faz isso:** o ReplicaSet controller (parte do kube-controller-manager).

**Pergunta provável do professor:** "Como funciona o self-healing?"
**Resposta:** O controlador compara o estado desejado com o atual. Se há menos Pods que o desejado, cria novos. Se há mais, remove. Loop contínuo.

### 4.5. Scaling e HPA

**Escalonamento horizontal:** aumentar/diminuir o número de réplicas (Pods).
**Escalonamento vertical:** aumentar/diminuir recursos (CPU, memória) de um Pod.

**HPA** faz escalonamento horizontal automático:
- Monitora o uso de CPU.
- Se ultrapassa o limite definido, cria mais réplicas.
- Se cai, reduz.

**Requer o `metrics-server` instalado no cluster** (no Minikube, é um addon).

**Pergunta provável do professor:** "O que é o HPA?"
**Resposta:** Horizontal Pod Autoscaler. Ajusta automaticamente o número de réplicas com base em métricas como CPU.

### 4.6. Diferença entre ClusterIP, NodePort e LoadBalancer

| Tipo | Onde expõe | Uso |
|------|-----------|-----|
| ClusterIP | Só dentro do cluster | Comunicação interna |
| NodePort | Porta específica do nó (30000-32767) | Acesso externo simples |
| LoadBalancer | Balanceador externo | Produção em nuvem |

No seu projeto, você vai usar **NodePort** para acessar a API de fora do cluster.

---

## 5. Passo a passo

### 5.1. Criar a estrutura de pastas

```bash
cd ~/projeto-a3
mkdir -p k8s
cd k8s
```

### 5.2. Criar o arquivo `namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: daemon-lab
```

**Explicação:**
- `apiVersion: v1` → versão da API do Kubernetes.
- `kind: Namespace` → tipo do recurso.
- `name: daemon-lab` → nome do namespace.

### 5.3. Criar os arquivos de Deployment e Service

**`api-deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  namespace: daemon-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: process-api
  template:
    metadata:
      labels:
        app: process-api
    spec:
      hostPID: true
      containers:
      - name: api
        image: process-api:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 8000
```

**Explicação:**
- `replicas: 2` → queremos 2 Pods rodando.
- `selector.matchLabels.app: process-api` → o Deployment gerencia Pods com esse label.
- `template.metadata.labels.app: process-api` → o label dos Pods criados.
- `hostPID: true` → compartilha o namespace de PID do host.
- `image: process-api:latest` → imagem da API.
- `imagePullPolicy: Never` → use a imagem local (não baixe do Docker Hub).
- `containerPort: 8000` → porta que o container expõe.

**`api-service.yaml`:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: daemon-lab
spec:
  type: NodePort
  selector:
    app: process-api
  ports:
  - port: 8000
    targetPort: 8000
    nodePort: 30080
```

**Explicação:**
- `type: NodePort` → expõe o serviço em uma porta do nó.
- `selector.app: process-api` → encontra os Pods com esse label.
- `port: 8000` → porta do Service.
- `targetPort: 8000` → porta do container.
- `nodePort: 30080` → porta exposta no nó (30000-32767).

**`nginx-deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: daemon-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx-cron:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 80
```

**`cron-deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cron-deployment
  namespace: daemon-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cron
  template:
    metadata:
      labels:
        app: cron
    spec:
      containers:
      - name: cron
        image: cron-daemon:latest
        imagePullPolicy: Never
```

### 5.4. Criar o arquivo `hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: daemon-lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

**Explicação:**
- `scaleTargetRef` → qual Deployment o HPA vai escalar.
- `minReplicas: 2` → mínimo de réplicas.
- `maxReplicas: 10` → máximo de réplicas.
- `averageUtilization: 50` → alvo de 50% de uso de CPU.

### 5.5. Fazer o deploy no Minikube

> ⚠️ **Este passo depende da Pessoa 1 e da Pessoa 2.** Você precisa das imagens `process-api:latest`, `nginx-cron:latest` e `cron-daemon:latest` buildadas.

**Se as imagens ainda não estiverem prontas**, use imagens genéricas temporariamente para testar os YAMLs:
- Troque `process-api:latest` por `nginx:latest` no `api-deployment.yaml`.
- Troque `nginx-cron:latest` por `nginx:latest` no `nginx-deployment.yaml`.
- Troque `cron-daemon:latest` por `alpine:latest` no `cron-deployment.yaml`.

Quando as imagens reais chegarem, troque de volta.

**Passo 1: Iniciar o Minikube**
```bash
minikube start --driver=docker
```

**Passo 2: Carregar as imagens no Minikube**
```bash
minikube image load process-api:latest
minikube image load nginx-cron:latest
minikube image load cron-daemon:latest
```

**Passo 3: Aplicar os YAMLs**
```bash
cd ~/projeto-a3/k8s
kubectl apply -f namespace.yaml
kubectl apply -f api-deployment.yaml
kubectl apply -f api-service.yaml
kubectl apply -f nginx-deployment.yaml
kubectl apply -f cron-deployment.yaml
```

**Passo 4: Verificar**
```bash
kubectl get pods -n daemon-lab
kubectl get svc -n daemon-lab
kubectl get deployments -n daemon-lab
```

**Passo 5: Acessar a API**
```bash
curl http://$(minikube ip):30080/processes
```

Ou:
```bash
minikube service api-service -n daemon-lab
```

### 5.6. Demonstrar self-healing

**Passo 1:** Ver os Pods rodando
```bash
kubectl get pods -n daemon-lab
```

**Passo 2:** Deletar um Pod da API
```bash
kubectl delete pod <nome-de-um-pod-api> -n daemon-lab
```

**Passo 3:** Observar a recriação automática
```bash
kubectl get pods -n daemon-lab -w
```

Você vai ver um novo Pod sendo criado em segundos.

**Passo 4:** Gravar um vídeo dessa demonstração.

### 5.7. Demonstrar scaling

**Passo 1:** Habilitar o metrics-server
```bash
minikube addons enable metrics-server
```

**Passo 2:** Aplicar o HPA
```bash
kubectl apply -f hpa.yaml
```

**Passo 3:** Ver o HPA (pode demorar 1-2 min para mostrar métricas)
```bash
kubectl get hpa -n daemon-lab -w
```

**Passo 4:** Gerar carga na API (em outro terminal)
```bash
while true; do curl http://$(minikube ip):30080/processes; done
```

**Passo 5:** Observar o escalonamento
```bash
kubectl get pods -n daemon-lab -w
```

Você vai ver o número de réplicas da API aumentando.

**Passo 6:** Parar a carga (Ctrl+C no terminal que está gerando carga).
Depois de alguns minutos, o HPA vai reduzir as réplicas de volta.

**Passo 7:** Gravar um vídeo dessa demonstração.

### 5.8. Tirar prints

Tire prints de:
1. `kubectl get pods -n daemon-lab` (Pods rodando)
2. `kubectl get svc -n daemon-lab` (Service)
3. `kubectl get hpa -n daemon-lab` (HPA)
4. Pod sendo recriado após self-healing
5. Réplicas aumentando após scaling
6. `curl http://$(minikube ip):30080/processes` (API respondendo)

Salve em `docs/evidencias/pessoa-3/`.

### 5.9. Documentar resultados

Crie uma tabela com os resultados. Exemplo:

| Métrica | Antes | Depois |
|---------|-------|--------|
| Réplicas da API | 2 | 5 |
| Tempo de recriação do Pod | — | 15s |
| Uso de CPU durante carga | 20% | 85% |

Esses dados vão para o artigo (Pessoa 4).

### 5.10. Subir para o GitHub

```bash
cd ~/projeto-a3
git status
git checkout -b feature/k8s
git add k8s/
git commit -m "Adiciona YAMLs do Kubernetes e demonstrações"
git push origin feature/k8s
```

Depois, no GitHub:
1. Abra o repositório.
2. Clique em **Compare & pull request**.
3. Escreva uma descrição curta.
4. Clique em **Create pull request**.
5. Avise no grupo para alguém revisar.

**Avise a Pessoa 4** que os resultados estão prontos para o artigo.

---

## 6. Código completo (resumo dos arquivos)

### `k8s/namespace.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: daemon-lab
```

### `k8s/api-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  namespace: daemon-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: process-api
  template:
    metadata:
      labels:
        app: process-api
    spec:
      hostPID: true
      containers:
      - name: api
        image: process-api:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 8000
```

### `k8s/api-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  namespace: daemon-lab
spec:
  type: NodePort
  selector:
    app: process-api
  ports:
  - port: 8000
    targetPort: 8000
    nodePort: 30080
```

### `k8s/nginx-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: daemon-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx-cron:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 80
```

### `k8s/cron-deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cron-deployment
  namespace: daemon-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cron
  template:
    metadata:
      labels:
        app: cron
    spec:
      containers:
      - name: cron
        image: cron-daemon:latest
        imagePullPolicy: Never
```

### `k8s/hpa.yaml`
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
  namespace: daemon-lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

---

## 7. Erros comuns e soluções

| Erro | Causa | Solução |
|------|-------|---------|
| `ImagePullBackOff` | Imagem não carregada no Minikube | Rodar `minikube image load <imagem>` |
| `CrashLoopBackOff` | Container com erro | Ver logs: `kubectl logs <pod> -n daemon-lab` |
| `Pending` | Falta de recursos | Ver `kubectl describe pod <pod> -n daemon-lab` |
| `ErrImageNeverPull` | `imagePullPolicy: Never` mas imagem não existe | Carregar a imagem no Minikube |
| HPA não escala | metrics-server desativado | `minikube addons enable metrics-server` |
| HPA mostra `<unknown>` | Métricas ainda não disponíveis | Esperar 1-2 minutos |
| `minikube start` falha | Docker Desktop fechado | Abrir o Docker Desktop e esperar |
| Pod não recriado | Deployment não gerenciando | Verificar `kubectl get deployments -n daemon-lab` |
| `connection refused` no curl | Service não está exposto ou Pod não está pronto | Verificar `kubectl get svc` e `kubectl get pods` |
| `namespace not found` | Namespace não foi aplicado | Rodar `kubectl apply -f namespace.yaml` primeiro |

---

## 8. Próximos passos

Depois de terminar:

1. Avise no grupo que os YAMLs estão prontos e as demonstrações foram feitas.
2. Envie os prints e vídeos para a Pessoa 4.
3. Envie a tabela de resultados para a Pessoa 4 (vai para o artigo).
4. Se a Pessoa 1 ou a Pessoa 2 mudarem as imagens, refaça o deploy.
5. Guarde os prints para o artigo e os slides.
