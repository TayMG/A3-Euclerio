# Guia de Comandos — Apresentação A3 (Tema 1)


**Pré-requisitos:** Docker Desktop aberto e WSL2/Ubuntu configurado. O terminal deve estar na raiz do projeto (`A3-Euclerio` ou `projeto-a3`).

---

## 1. Preparação e Build (Docker)

Caso as imagens não estejam prontas na máquina:

```bash
cd api && docker build -t process-api:latest . && cd ..
cd nginx && docker build -t nginx-cron:latest . && cd ..
cd cron && docker build -t cron-daemon:latest . && cd ..
```

---

## 2. Deploy no Kubernetes (Minikube)

Iniciar o cluster, carregar as imagens e aplicar os manifestos:

```bash
# Iniciar o cluster
minikube start --driver=docker

# Injetar as imagens locais no Minikube
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

---

## 3. Verificação Básica

```bash
# Verificar os 5 Pods rodando
kubectl get pods -n daemon-lab

# Verificar os Services (NodePort 30080 e 30081)
kubectl get svc -n daemon-lab

# Verificar o HPA configurado
kubectl get hpa -n daemon-lab
```

---

## 4. Acesso via Navegador e Terminal

Para acessar a API e o Nginx no navegador do Windows:

```bash
# Abrir a API (Deixe o terminal aberto)
minikube service api-service -n daemon-lab

# Abrir o Nginx (Deixe outro terminal aberto)
minikube service nginx-service -n daemon-lab
```

**Teste Rápido no Terminal (sem navegador):**
Substitua `<PORTA_DO_TUNEL>` pela porta gerada no comando acima (ex: 45153).
```bash
# Listar processos (teste do hostPID)
curl http://127.0.0.1:<PORTA_DO_TUNEL>/processes

# Ler o arquivo do cron dentro do Pod do Nginx
kubectl exec -it <nome-do-pod-nginx> -n daemon-lab -- cat /usr/share/nginx/html/cron.txt
```

---

## 5. Testes Dinâmicos (Demonstração ao Vivo)

### Teste de Self-healing (Matar Pod)

```bash
# 1. Copie o nome de um pod da API
kubectl get pods -n daemon-lab

# 2. Delete o pod (substitua <nome-do-pod>)
kubectl delete pod <nome-do-pod> -n daemon-lab

# 3. Veja ele renascendo em tempo real
kubectl get pods -n daemon-lab -w
# (Aperte Ctrl+C para parar de assistir)
```

### Teste de HPA (Escalonamento)

```bash
# 1. Ativar o medidor de CPU
minikube addons enable metrics-server
kubectl apply -f k8s/hpa.yaml

# 2. Verificar se o metrics-server está lendo a CPU
kubectl top pods -n daemon-lab

# 3. Em outro terminal, gere carga na API (Troque <PORTA_DO_TUNEL> pelo número do passo 4)
while true; do curl -s http://127.0.0.1:<PORTA_DO_TUNEL>/processes > /dev/null; done

# 4. Em outro terminal, assista o HPA criar réplicas
kubectl get hpa -n daemon-lab -w
```

---

## 6. Modificações ao Vivo (Arguição)

Comandos para responder aos desafios do professor sem precisar abrir arquivos YAML.

### Mudar a porcentagem de CPU do HPA (ex: de 50% para 70%)
```bash
kubectl patch hpa api-hpa -n daemon-lab --type=json -p='[{"op": "replace", "path": "/spec/metrics/0/resource/target/averageUtilization", "value": 70}]'
```

### Mudar a quantidade de réplicas manualmente
```bash
kubectl scale deployment api-deployment --replicas=4 -n daemon-lab
```
*Atenção: O HPA vai brigar com você se a CPU estiver baixa. Para que as réplicas se mantenham, altere o `minReplicas` do HPA no comando abaixo.*

### Impedir o HPA de derrubar as réplicas manuais
```bash
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"minReplicas":4}}'
```

### Resetar as configurações ao final da apresentação
```bash
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"minReplicas":2}}'
kubectl patch hpa api-hpa -n daemon-lab --type=json -p='[{"op": "replace", "path": "/spec/metrics/0/resource/target/averageUtilization", "value": 50}]'
```

---

## 7. Diagnóstico (Se der erro)

```bash
# Pod travado em Pending ou ImagePullBackOff
kubectl describe pod <nome-do-pod> -n daemon-lab

# API dando erro 500 (ver logs do Python)
kubectl logs <nome-do-pod-api> -n daemon-lab
```

---

## 8. Reset Total (Fim da Apresentação)

Para limpar o ambiente e não consumir recursos do computador:

```bash
kubectl delete namespace daemon-lab
minikube delete
```
