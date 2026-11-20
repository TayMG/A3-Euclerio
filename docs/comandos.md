```markdown
# Guia de Comandos — Apresentação A3 (Tema 1)

> **Pré-requisitos:** Docker Desktop aberto e WSL2/Ubuntu configurado. O terminal deve estar na raiz do projeto (`A3-Euclerio` ou `projeto-a3`).
>
> **Alinhamento com `docs/interfaces.md`:** este guia usa os nomes exatos definidos nas interfaces (imagens, namespace, portas, labels, endpoints). O projeto usa `imagePullPolicy: Never` em todos os YAMLs.

---

## 0. Pré-verificação do ambiente

Antes de começar a apresentação, garanta que tudo está no lugar:

```bash
minikube status
kubectl config current-context
kubectl get nodes
docker ps
```

Se o `minikube status` não mostrar `Running`, execute `minikube start --driver=docker`.

### 0.1. Confirmar nomes e valores exatos (evita falhas ao vivo)

O guia assume nomes fixos. Confirme-os **antes** de apresentar:

```bash
# Nome do namespace
kubectl get ns | grep daemon-lab

# Nomes dos Deployments
kubectl get deploy -n daemon-lab -o custom-columns=NAME:.metadata.name

# Nome do container dentro do Deployment da API (usado na secção 6.5)
kubectl get deploy api-deployment -n daemon-lab -o jsonpath='{.spec.template.spec.containers[*].name}'

# Nome do HPA e valores originais (usados nas secções 6.1, 6.2 e 6.9)
kubectl get hpa -n daemon-lab
kubectl get hpa api-hpa -n daemon-lab -o yaml | grep -E "name:|minReplicas|averageUtilization|targetCPUUtilizationPercentage|apiVersion"

# Labels reais dos pods (usados na secção 5.1)
kubectl get pods -n daemon-lab --show-labels
```

Se algum valor diferir do que o guia assume, ajuste o comando correspondente antes de apresentar.

---

## 1. Preparação e Build (Docker)

Caso as imagens não estejam prontas na máquina:

```bash
cd api   && docker build -t process-api:latest . && cd ..
cd nginx && docker build -t nginx-cron:latest . && cd ..
cd cron  && docker build -t cron-daemon:latest . && cd ..
```

Verificação rápida:

```bash
docker images | grep -E "process-api|nginx-cron|cron-daemon"
```

> **Nota sobre `imagePullPolicy: Never`:** os YAMLs do projeto usam `imagePullPolicy: Never` (definido nas interfaces). Isto significa que o Minikube **não tenta fazer pull** da imagem — se ela não estiver carregada no cluster, o pod fica em `ErrImageNeverPull`. Após qualquer rebuild, é **obrigatório** recarregar a imagem:
>
> ```bash
> minikube image load process-api:latest
> kubectl rollout restart deployment/api-deployment -n daemon-lab
> ```

---

## 2. Deploy no Kubernetes (Minikube)

Iniciar o cluster, carregar as imagens e aplicar os manifestos:

```bash
# Iniciar o cluster
minikube start --driver=docker

# Injetar as imagens locais no Minikube (obrigatório por causa do imagePullPolicy: Never)
minikube image load process-api:latest
minikube image load nginx-cron:latest
minikube image load cron-daemon:latest

# Aplicar os YAMLs
cd k8s
kubectl apply -f namespace.yaml
kubectl apply -f api-deployment.yaml
kubectl apply -f api-service.yaml
kubectl apply -f nginx-deployment.yaml
kubectl apply -f nginx-service.yaml
kubectl apply -f cron-deployment.yaml
cd ..
```

### 2.1. Ativar o metrics-server e aplicar o HPA

O HPA **depende** do `metrics-server`. Sem ele, o HPA fica com `<unknown>`.

```bash
# Ativar o medidor de CPU
minikube addons enable metrics-server

# Esperar o metrics-server ficar pronto (pode demorar 30-60s)
kubectl wait --for=condition=Available deployment/metrics-server -n kube-system --timeout=120s
sleep 20

# Aplicar o HPA
kubectl apply -f k8s/hpa.yaml
```

### 2.2. Aguardar os deployments estarem prontos

```bash
kubectl wait --for=condition=available deployment/api-deployment   -n daemon-lab --timeout=120s
kubectl wait --for=condition=available deployment/nginx-deployment -n daemon-lab --timeout=120s
kubectl wait --for=condition=available deployment/cron-deployment  -n daemon-lab --timeout=120s
```

### 2.3. Criar uma imagem "v2" para o demo de rollback

O demo da secção 6.5 faz `kubectl set image ... <container>=process-api:v2`. Essa imagem tem de existir no Minikube, senão o pod fica em `ErrImageNeverPull`.

```bash
# Criar uma "v2" a partir da imagem atual (apenas para o demo de rollback)
docker tag process-api:latest process-api:v2
minikube image load process-api:v2
```

---

## 3. Verificação Básica

```bash
# Verificar todos os recursos do namespace
kubectl get all -n daemon-lab

# Verificar os Pods (READY deve ser 1/1)
kubectl get pods -n daemon-lab -o wide

# Verificar os Services (NodePort 30080 e 30081)
kubectl get svc -n daemon-lab

# Verificar os Endpoints (garante que o Service está ligado aos Pods)
kubectl get endpoints -n daemon-lab

# Confirmar os labels reais dos pods (usados nos demos de self-healing)
kubectl get pods -n daemon-lab --show-labels

# Verificar o HPA (deve mostrar targets de CPU após ~1 min)
kubectl get hpa -n daemon-lab
kubectl describe hpa api-hpa -n daemon-lab
```

> **Nota:** se o HPA mostrar `<unknown>` no `TARGETS`, aguarde ~1 minuto após o `metrics-server` ficar pronto. Se persistir, execute `kubectl top pods -n daemon-lab` para confirmar que as métricas estão a ser lidas.

---

## 4. Acesso via Navegador e Terminal

Para acessar a API e o Nginx no navegador do Windows:

```bash
# Abrir a API (deixe o terminal aberto)
minikube service api-service -n daemon-lab

# Abrir o Nginx (deixe outro terminal aberto)
minikube service nginx-service -n daemon-lab
```

**Teste rápido no terminal (sem navegador):**
Substitua `<PORTA>` pela porta gerada no comando acima (ex: `45153`).

### 4.1. Endpoints da API (definidos nas interfaces)

```bash
# Listar todos os processos
curl http://127.0.0.1:<PORTA>/processes

# Inspecionar um processo pelo PID
curl http://127.0.0.1:<PORTA>/processes/1

# Iniciar um novo processo (parâmetro command)
curl -X POST "http://127.0.0.1:<PORTA>/processes/start?command=sleep%2060"

# Encerrar um processo pelo PID (parâmetro force opcional, padrão false)
curl -X DELETE "http://127.0.0.1:<PORTA>/processes/<PID>?force=true"

# Documentação automática (Swagger) — abrir no navegador
# http://127.0.0.1:<PORTA>/docs
```

### 4.2. Acesso ao Nginx e ao Cron

```bash
# Ler o ficheiro escrito pelo cron dentro do Pod do Nginx
kubectl exec -it deploy/nginx-deployment -n daemon-lab -- cat /usr/share/nginx/html/cron.txt

# Confirmar que o cron está a correr e a gerar logs
kubectl logs deploy/cron-deployment -n daemon-lab --tail=20
```

---

## 5. Testes Dinâmicos (Demonstração ao Vivo)

### 5.1. Teste de Self-healing (Matar Pod)

```bash
# 1. Copie o nome de um pod da API
kubectl get pods -n daemon-lab

# 2. Delete o pod (substitua <nome-do-pod>)
kubectl delete pod <nome-do-pod> -n daemon-lab

# 3. Veja-o a renascer em tempo real
kubectl get pods -n daemon-lab -w
# (Ctrl+C para parar)
```

**Self-healing por label nos três serviços:**

```bash
# Confirmar os labels reais antes (evita "No resources found")
kubectl get pods -n daemon-lab --show-labels

# Matar o pod da API (label definido nas interfaces: app: process-api)
kubectl delete pod -l app=process-api -n daemon-lab

# Matar o pod do Nginx (label definido nas interfaces: app: nginx)
kubectl delete pod -l app=nginx -n daemon-lab

# Matar o pod do Cron (label definido nas interfaces: app: cron)
kubectl delete pod -l app=cron -n daemon-lab

# Alternativa robusta: obter o label diretamente do selector do Deployment
kubectl delete pod -n daemon-lab -l app=$(kubectl get deploy api-deployment -n daemon-lab -o jsonpath='{.spec.selector.matchLabels.app}')

# Observar a recuperação
kubectl get pods -n daemon-lab -w
```

### 5.2. Teste de HPA (Escalonamento)

```bash
# 1. Confirmar que o metrics-server está a ler a CPU
kubectl top pods -n daemon-lab

# 2. Num terminal, gerar carga na API (troque <PORTA>)
while true; do curl -s http://127.0.0.1:<PORTA>/processes > /dev/null; done

# 3. Noutro terminal, assistir o HPA a criar réplicas
kubectl get hpa -n daemon-lab -w

# 4. (Opcional) Ver as réplicas a subir
kubectl get pods -n daemon-lab -w
```

Para parar a carga: `Ctrl+C` no terminal do `while`.

**Plano B** (se o `while` não gerar carga suficiente para o HPA reagir):

```bash
kubectl run stress --image=busybox -n daemon-lab --restart=Never -- \
  sh -c "while true; do wget -q -O- http://api-service:8000/processes; done"
kubectl get hpa -n daemon-lab -w
# Limpar depois
kubectl delete pod stress -n daemon-lab
```

---

## 6. Modificações ao Vivo (Arguição)

Comandos para responder aos desafios do professor **sem abrir ficheiros YAML**.

### 6.1. HPA — mudar a percentagem de CPU (ex: 50% → 70%)

> **Atenção à versão da API.** O JSON path usado depende de `autoscaling/v2`. Confirme antes:

```bash
kubectl get hpa api-hpa -n daemon-lab -o yaml | grep "apiVersion"
```

Se for `autoscaling/v2`:

```bash
# Aplicar
kubectl patch hpa api-hpa -n daemon-lab --type=json \
  -p='[{"op": "replace", "path": "/spec/metrics/0/resource/target/averageUtilization", "value": 70}]'

# Verificar
kubectl get hpa -n daemon-lab

# Reverter
kubectl patch hpa api-hpa -n daemon-lab --type=json \
  -p='[{"op": "replace", "path": "/spec/metrics/0/resource/target/averageUtilization", "value": 50}]'
```

Se for `autoscaling/v1`:

```bash
# Aplicar
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"targetCPUUtilizationPercentage":70}}'

# Verificar
kubectl get hpa -n daemon-lab

# Reverter
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"targetCPUUtilizationPercentage":50}}'
```

### 6.2. HPA — mudar o `minReplicas`

```bash
# Aplicar
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"minReplicas":4}}'

# Verificar
kubectl get hpa -n daemon-lab

# Reverter (assumindo minReplicas=2 no projeto — confirmar na secção 0.1)
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"minReplicas":2}}'
```

### 6.3. Scale manual do Deployment

> **Atenção:** o HPA vai "brigar" com o scale manual se a CPU estiver baixa. Para o scale **permanecer**, aumente também o `minReplicas` (ver 6.2).

```bash
# Aplicar
kubectl scale deployment api-deployment --replicas=4 -n daemon-lab

# Verificar
kubectl get pods -n daemon-lab -w

# Reverter
kubectl scale deployment api-deployment --replicas=2 -n daemon-lab
```

### 6.4. Alterar variável de ambiente (rolling restart automático)

```bash
# Aplicar
kubectl set env deployment/api-deployment MINHA_VAR=novo -n daemon-lab

# Verificar (o Deployment faz rolling update sozinho)
kubectl rollout status deployment/api-deployment -n daemon-lab

# Reverter
kubectl set env deployment/api-deployment MINHA_VAR- -n daemon-lab
```

### 6.5. Trocar a imagem e reverter (rolling update / rollback)

Requer a imagem `process-api:v2` criada no passo 2.3. O nome do container (`api` abaixo) deve ser confirmado na secção 0.1.

```bash
# Aplicar nova imagem
kubectl set image deployment/api-deployment api=process-api:v2 -n daemon-lab

# Verificar o rollout
kubectl rollout status deployment/api-deployment -n daemon-lab

# Ver histórico
kubectl rollout history deployment/api-deployment -n daemon-lab

# Reverter
kubectl rollout undo deployment/api-deployment -n daemon-lab
```

### 6.6. Rolling restart (reiniciar pods sem mudar nada)

```bash
kubectl rollout restart deployment/api-deployment -n daemon-lab
kubectl rollout status  deployment/api-deployment -n daemon-lab
```

### 6.7. ConfigMap — criar, montar como env e reverter

```bash
# Aplicar
kubectl create configmap api-config -n daemon-lab \
  --from-literal=LOG_LEVEL=debug

# Verificar
kubectl get configmap api-config -n daemon-lab -o yaml

# Montar como variável de ambiente no Deployment (sem editar YAML)
kubectl set env deployment/api-deployment -n daemon-lab --from=configmap/api-config
kubectl rollout status deployment/api-deployment -n daemon-lab

# Reverter
kubectl set env deployment/api-deployment -n daemon-lab --from=configmap/api-config-
kubectl delete configmap api-config -n daemon-lab
```

### 6.8. Probes e Resource Limits (explicação + inspeção)

```bash
# Mostrar liveness/readiness probes atuais
kubectl describe pod <nome-do-pod-api> -n daemon-lab | grep -A 5 -i "liveness\|readiness"

# Mostrar resource requests e limits
kubectl describe pod <nome-do-pod-api> -n daemon-lab | grep -A 5 -i "limits\|requests"

# Consumo real
kubectl top pods -n daemon-lab
```

### 6.9. Reset final das modificações ao vivo

```bash
# HPA: repor valores originais (confirmar em 0.1 se não forem 2 e 50%)
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"minReplicas":2}}'
kubectl patch hpa api-hpa -n daemon-lab --type=json \
  -p='[{"op": "replace", "path": "/spec/metrics/0/resource/target/averageUtilization", "value": 50}]'

# Deployment: repor réplicas originais
kubectl scale deployment api-deployment --replicas=2 -n daemon-lab

# Rollback de imagem, se alterada
kubectl rollout undo deployment/api-deployment -n daemon-lab
```

---

## 7. Diagnóstico (se der erro)

### 7.1. Pod travado em Pending, ImagePullBackOff, ErrImageNeverPull ou CrashLoopBackOff

```bash
kubectl describe pod <nome-do-pod> -n daemon-lab
kubectl logs <nome-do-pod> -n daemon-lab
kubectl logs <nome-do-pod> -n daemon-lab --previous   # logs do crash anterior
```

> **`ErrImageNeverPull`:** a imagem não está carregada no Minikube e o `imagePullPolicy: Never` impede o pull. Corrija com:
>
> ```bash
> minikube image load <nome-da-imagem>:latest
> kubectl rollout restart deployment/<nome-do-deployment> -n daemon-lab
> ```

### 7.2. API a devolver erro 500

```bash
kubectl logs <nome-do-pod-api> -n daemon-lab --tail=100
```

### 7.3. Service sem resposta

```bash
# Confirmar que o Service tem endpoints
kubectl get endpoints -n daemon-lab

# Se estiver vazio, os labels do Service não batem com os do Pod
kubectl describe svc api-service -n daemon-lab
kubectl get pods -n daemon-lab --show-labels
```

### 7.4. Ver eventos recentes (diagnóstico rápido)

```bash
kubectl get events -n daemon-lab --sort-by=.lastTimestamp | tail -20
```

### 7.5. HPA com `<unknown>`

```bash
kubectl top pods -n daemon-lab                # métricas a funcionar?
kubectl get apiservice v1beta1.metrics.k8s.io # metrics-server registado?
kubectl logs -n kube-system deploy/metrics-server --tail=50
```

---

## 8. Reset Total (fim da apresentação)

> **Atenção:** `kubectl delete namespace` apaga **tudo** do projeto, mas mantém o cluster. `minikube delete` apaga o **cluster inteiro** (todas as imagens carregadas incluídas).

```bash
# Apagar apenas o projeto (mantém o Minikube)
kubectl delete namespace daemon-lab

# Apagar o cluster completo (usar só no fim de tudo)
minikube delete
```

Se quiser apenas **parar** o cluster sem o apagar:

```bash
minikube stop
```
