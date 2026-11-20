# Guia de Comandos — Apresentação A3 (Tema 1)

> **Pré-requisitos:** Docker Desktop aberto, WSL2/Ubuntu configurado e terminal na raiz do repositório (`A3-Euclerio`).
>
> **Execute tudo no terminal do Ubuntu/WSL.** Os comandos usam `grep`, `while`, `&` e continuação de linha com `\`, que não funcionam igual no PowerShell.
>
> **Alinhamento:** nomes de imagens, namespace, portas, labels e endpoints seguem `docs/interfaces.md`.
>
> **Regra de ouro:** tudo o que pode baixar (imagens base, metrics-server, ISO do Minikube) deve ser preparado **antes** da apresentação. Durante a demo, use apenas o que já está no cluster.
>
> **Leitura rápida:** cada bloco da secção 6 começa com **O que faz / Não muda**. `scale nginx` mexe só no Nginx; `patch hpa` mexe na política (mínimo, máximo ou alvo), não na imagem; `set env`/`set image`/`set resources` mexem no pod da API; `patch svc` mexe só na porta/tipo do Service.

---

## 0. Pré-verificação

Se for a primeira execução, pule para a secção 1 e volte aqui depois do deploy. Se o projeto já está no cluster:

```bash
docker ps
minikube status
kubectl config current-context
kubectl get nodes
```

Confirme os recursos do projeto (sem falhar se ainda não existirem):

```bash
if kubectl get namespace daemon-lab >/dev/null 2>&1; then
  kubectl get deploy -n daemon-lab
  kubectl get pods -n daemon-lab --show-labels
  kubectl get svc -n daemon-lab
  kubectl get hpa -n daemon-lab
else
  echo "Namespace daemon-lab ainda não existe. Siga para a secção 1."
fi
```

Valores que o guia assume (confirme antes de apresentar):

```bash
kubectl get ns daemon-lab
kubectl get deploy -n daemon-lab -o custom-columns=NAME:.metadata.name
kubectl get deploy api-deployment -n daemon-lab -o jsonpath='{.spec.template.spec.containers[*].name}'
kubectl get hpa api-hpa -n daemon-lab -o yaml | grep -E "apiVersion|minReplicas|maxReplicas|averageUtilization"
kubectl get pods -n daemon-lab --show-labels
```

---

## 1. Build das imagens (fazer antes da apresentação)

Só é necessário se as imagens ainda não estiverem prontas. Durante a demo, pule esta secção se `docker images` já mostrar as três.

```bash
cd api   && docker build -t process-api:latest . && cd ..
cd nginx && docker build -t nginx-cron:latest . && cd ..
cd cron  && docker build -t cron-daemon:latest . && cd ..
```

Verificação:

```bash
docker images | grep -E "process-api|nginx-cron|cron-daemon"
```

> **Nota sobre `imagePullPolicy: Never`:** os YAMLs usam `Never`. O Minikube não tenta fazer pull; se a imagem não estiver carregada, o pod fica em `ErrImageNeverPull`. Após qualquer rebuild, recarregue a imagem e reinicie o Deployment:
>
> ```bash
> minikube image load process-api:latest
> kubectl rollout restart deployment/api-deployment -n daemon-lab
> ```

---

## 2. Deploy no Minikube

```bash
minikube start --driver=docker
kubectl config use-context minikube
```

Carregue as imagens locais no cluster:

```bash
minikube image load process-api:latest
minikube image load nginx-cron:latest
minikube image load cron-daemon:latest
```

Aplique os manifestos:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Active namespace/daemon-lab --timeout=60s

kubectl apply -f k8s/api-deployment.yaml
kubectl apply -f k8s/api-service.yaml
kubectl apply -f k8s/nginx-deployment.yaml
kubectl apply -f k8s/nginx-service.yaml
kubectl apply -f k8s/cron-deployment.yaml
```

### 2.1. Metrics-server e HPA

O HPA depende do `metrics-server`. Ative antes da demo, se possível; durante a demo, apenas confirme.

```bash
minikube addons enable metrics-server
kubectl wait --for=condition=Available deployment/metrics-server -n kube-system --timeout=180s
kubectl wait --for=condition=Available apiservice/v1beta1.metrics.k8s.io --timeout=120s || true
kubectl apply -f k8s/hpa.yaml
```

### 2.2. Aguardar os Deployments

```bash
kubectl wait --for=condition=Available deployment/api-deployment   -n daemon-lab --timeout=180s
kubectl wait --for=condition=Available deployment/nginx-deployment -n daemon-lab --timeout=180s
kubectl wait --for=condition=Available deployment/cron-deployment  -n daemon-lab --timeout=180s
```

### 2.3. Imagem `v2` para o demo de rollback

A imagem `v2` é apenas uma tag da imagem atual; serve para demonstrar `set image` e `rollout undo` sem baixar nada.

```bash
docker tag process-api:latest process-api:v2
minikube image load process-api:v2
```

---

## 3. Verificação básica

```bash
kubectl get all -n daemon-lab
kubectl get pods -n daemon-lab -o wide
kubectl get svc -n daemon-lab
kubectl get endpointslices -n daemon-lab
kubectl get pods -n daemon-lab --show-labels
kubectl get hpa -n daemon-lab
kubectl describe hpa api-hpa -n daemon-lab
```

> Se o HPA mostrar `<unknown>`, aguarde 1–2 minutos após o metrics-server ficar pronto e confirme com `kubectl top pods -n daemon-lab`.

---

## 4. Acesso à API e ao Nginx

Abra cada serviço num terminal separado e deixe o terminal aberto (o `minikube service` mantém o túnel):

```bash
minikube service api-service -n daemon-lab
minikube service nginx-service -n daemon-lab
```

**Plano B** se o `minikube service` falhar:

```bash
kubectl port-forward svc/api-service 8000:8000 -n daemon-lab
kubectl port-forward svc/nginx-service 8080:80 -n daemon-lab
```

### 4.1. Endpoints da API

Substitua `<PORTA>` pela porta do túnel (ex.: `45153`) ou use `8000` se estiver com port-forward.

```bash
# Listar processos (limite a saída para não encher o terminal)
curl -s http://127.0.0.1:<PORTA>/processes | head -c 1500

# Inspecionar um PID
curl -s http://127.0.0.1:<PORTA>/processes/1

# Iniciar um processo (use 600s para dar tempo de copiar o PID e encerrar)
curl -X POST "http://127.0.0.1:<PORTA>/processes/start?command=sleep%20600"

# Encerrar o PID devolvido (force é opcional, padrão false)
curl -X DELETE "http://127.0.0.1:<PORTA>/processes/<PID>?force=true"
```

Swagger: abra `http://127.0.0.1:<PORTA>/docs` no navegador.

### 4.2. Nginx e cron

O cron escreve uma vez por minuto. **Espere o primeiro tick** antes de mostrar; caso contrário, o ficheiro ainda não existe.

```bash
# Nginx + cron embutido
kubectl exec -it deploy/nginx-deployment -n daemon-lab -- cat /usr/share/nginx/html/cron.txt

# Cron isolado
kubectl exec -it deploy/cron-deployment -n daemon-lab -- cat /var/log/cron-task.log
```

> O Nginx tem 2 réplicas e cada pod tem o seu próprio `cron.txt`. O `kubectl exec deploy/...` escolhe um pod; o navegador pode cair no outro. Se quiser consistência, escolha um pod específico:
>
> ```bash
> kubectl get pods -l app=nginx -n daemon-lab
> kubectl exec -it <nome-do-pod-nginx> -n daemon-lab -- cat /usr/share/nginx/html/cron.txt
> ```

---

## 5. Resiliência e autoescala

### 5.1. Self-healing

**O que faz:** apaga um pod da API e mostra o Deployment a recriá-lo. **Não muda:** imagem, configuração, réplicas desejadas.

Apague um pod da API pelo nome (mais suave do que apagar todos por label) e acompanhe a recriação:

```bash
kubectl get pods -l app=process-api -n daemon-lab
kubectl delete pod <nome-do-pod-api> -n daemon-lab
kubectl get pods -l app=process-api -n daemon-lab -w
```

O Deployment mantém o número de réplicas desejado; o pod apagado é recriado automaticamente. Enquanto o novo pod sobe, os restantes continuam a atender.

> Se quiser demonstrar indisponibilidade, apague os pods por label:
>
> ```bash
> kubectl delete pods -l app=process-api -n daemon-lab
> ```
>
> Nesse caso há alguns segundos sem pods prontos. Use apenas se o objetivo for mostrar o impacto.

### 5.2. Carga para o HPA

**O que faz:** cria um pod que chama a API em loop para subir o CPU e o HPA escalar. **Não muda:** alvo, mínimo ou máximo do HPA.

O HPA mede o CPU **dos pods do `api-deployment`**. A forma mais simples de gerar carga sem baixar nada é criar um pod com a própria imagem do projeto (já carregada no Minikube) que chama o endpoint `/processes` em loop.

```bash
kubectl run stress -n daemon-lab \
  --image=process-api:latest \
  --image-pull-policy=Never \
  --restart=Never \
  --command -- python -c "
import urllib.request
while True:
    try:
        urllib.request.urlopen('http://api-service:8000/processes', timeout=5).read()
    except Exception:
        pass
"
```

> **Por que `--image-pull-policy=Never`:** a tag `latest` faz o servidor assumir `Always`; sem essa flag o pod fica em `ErrImagePull`/`ErrImagePullBackOff`, mesmo com a imagem presente no nó. `Never` garante que o kubelet use apenas a imagem local.

Acompanhe a escala em terminais separados:

```bash
kubectl get pods -l app=process-api -n daemon-lab -w
kubectl get hpa api-hpa -n daemon-lab -w
kubectl top pods -n daemon-lab
```

O HPA tem `minReplicas: 2`, `maxReplicas: 10` e alvo de **50%** de CPU sobre o request de `100m`. A subida costuma ocorrer em 30–90 s após o metrics-server ter dados.

Encerre a carga:

```bash
kubectl delete pod stress -n daemon-lab
```

**Scale-down:** o comportamento padrão do HPA usa `stabilizationWindowSeconds: 300` (5 minutos). Não prometa 1–2 min. Para uma demo mais rápida, reduza a janela **antes** de gerar carga:

```bash
kubectl patch hpa api-hpa -n daemon-lab --type=merge \
  -p '{"spec":{"behavior":{"scaleDown":{"stabilizationWindowSeconds":60}}}}'
```

Reverta depois com:

```bash
kubectl apply -f k8s/hpa.yaml
```

> Se o HPA não reagir, confirme `kubectl top pods -n daemon-lab`. Se retornar erro, o metrics-server ainda não está pronto — veja a secção 2.1.

---

## 6. Modificações ao vivo

Todos os comandos abaixo podem ser executados durante a arguição. Cada bloco tem o respetivo revert.

### 6.1. Escalar um Deployment

**O que faz:** muda o número de pods do **Nginx** (2 → 4). **Não muda:** API, HPA, imagens.

```bash
# Muda só o número de pods do Nginx (2 -> 4); não toca na API
kubectl scale deployment nginx-deployment -n daemon-lab --replicas=4
kubectl get pods -l app=nginx -n daemon-lab -w
```

Revert:

```bash
kubectl scale deployment nginx-deployment -n daemon-lab --replicas=2
```

> O `api-deployment` é gerido pelo HPA. Escalá-lo manualmente funciona por instantes, mas o HPA volta ao valor calculado. Para a API, altere `minReplicas`/`maxReplicas` (secção 6.2) em vez de `scale`.

### 6.2. Alterar o HPA

**O que faz:** muda a **política** do HPA da API (alvo de CPU, mínimo ou máximo). **Não muda:** a imagem nem o Service; o número de pods só reage depois.

Mudar o alvo de CPU para 70%:

```bash
# Muda o ALVO de CPU do HPA (50% -> 70%); não cria pods sozinho
kubectl patch hpa api-hpa -n daemon-lab --type=merge \
  -p '{"spec":{"metrics":[{"type":"Resource","resource":{"name":"cpu","target":{"type":"Utilization","averageUtilization":70}}}]}}'
kubectl describe hpa api-hpa -n daemon-lab
```

Mudar mínimo e máximo (são dois limites independentes):

```bash
# Muda o MÍNIMO de pods da API (piso = 3)
kubectl patch hpa api-hpa -n daemon-lab --type=merge \
  -p '{"spec":{"minReplicas":3}}'

# Muda o MÁXIMO de pods da API (teto = 8)
kubectl patch hpa api-hpa -n daemon-lab --type=merge \
  -p '{"spec":{"maxReplicas":8}}'

kubectl get hpa api-hpa -n daemon-lab
```

Revert:

```bash
kubectl apply -f k8s/hpa.yaml
```

> Se o `apply` não limpar um campo criado por `patch` (por causa da anotação `last-applied-configuration`), recrie o HPA:
>
> ```bash
> kubectl delete hpa api-hpa -n daemon-lab
> kubectl apply -f k8s/hpa.yaml
> ```

### 6.3. Variável de ambiente

**O que faz:** adiciona (ou remove, com `-`) uma variável no pod da **API** e reinicia os pods. **Não muda:** imagem, Service, HPA.

```bash
# Adiciona MINHA_VAR ao pod da API (dispara novo rollout)
kubectl set env deployment/api-deployment -n daemon-lab MINHA_VAR=teste
kubectl rollout status deployment/api-deployment -n daemon-lab
kubectl exec -it deploy/api-deployment -n daemon-lab -- printenv MINHA_VAR
```

Revert:

```bash
# Remove MINHA_VAR do pod da API (o '-' no fim remove, não faz parte do nome)
kubectl set env deployment/api-deployment -n daemon-lab MINHA_VAR-
kubectl rollout status deployment/api-deployment -n daemon-lab
```

> `set env` altera o pod template; por isso há sempre um novo rollout. O `-` no final remove a variável (não faz parte do nome).

### 6.4. Troca de imagem e rollback

**O que faz:** troca a imagem do contentor `api` e depois volta à revisão anterior. **Não muda:** configuração, Service, HPA.

A imagem `process-api:v2` foi criada na secção 2.3 como tag da mesma imagem local — não há download.

```bash
# Troca só a imagem do contentor 'api' (não mexe em env, Service nem HPA)
kubectl set image deployment/api-deployment api=process-api:v2 -n daemon-lab
kubectl rollout status deployment/api-deployment -n daemon-lab
kubectl rollout history deployment/api-deployment -n daemon-lab
kubectl get pods -l app=process-api -n daemon-lab -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

Revert:

```bash
kubectl rollout undo deployment/api-deployment -n daemon-lab
kubectl rollout status deployment/api-deployment -n daemon-lab
```

> `rollout undo` volta **uma** revisão. Se fizer vários ciclos, confirme a imagem final. Para garantir o estado original:
>
> ```bash
> kubectl set image deployment/api-deployment api=process-api:latest -n daemon-lab
> kubectl rollout status deployment/api-deployment -n daemon-lab
> ```

### 6.5. Rolling restart

**O que faz:** recria os pods da API com a mesma configuração (útil após recarregar a imagem `:latest`). **Não muda:** imagem, env, Service.

Reinicia os pods sem alterar a configuração (útil após recarregar uma imagem com a mesma tag):

```bash
kubectl rollout restart deployment/api-deployment -n daemon-lab
kubectl rollout status deployment/api-deployment -n daemon-lab
kubectl get pods -l app=process-api -n daemon-lab
```

O rollout é gradual: os pods antigos só são terminados depois de os novos ficarem Ready.

### 6.6. ConfigMap

**O que faz:** cria um ConfigMap e injeta as chaves como variáveis no pod da **API**. **Não muda:** imagem, Service, HPA.

Crie (ou atualize) o ConfigMap de forma idempotente e injete-o como variáveis de ambiente:

```bash
# Cria/atualiza o ConfigMap (idempotente, não falha se já existir)
kubectl create configmap api-config -n daemon-lab \
  --from-literal=LOG_LEVEL=debug \
  --dry-run=client -o yaml | kubectl apply -f -

# Injeta as chaves do ConfigMap como env no pod da API
kubectl set env deployment/api-deployment -n daemon-lab --from=configmap/api-config
kubectl rollout status deployment/api-deployment -n daemon-lab
kubectl exec -it deploy/api-deployment -n daemon-lab -- printenv LOG_LEVEL
```

Revert (a ordem importa):

```bash
# 1) remover a referência do pod template
kubectl set env deployment/api-deployment -n daemon-lab LOG_LEVEL-
kubectl rollout status deployment/api-deployment -n daemon-lab

# 2) só agora apagar o ConfigMap
kubectl delete configmap api-config -n daemon-lab
```

> **Nunca** use `kubectl set env ... --from=configmap/api-config-`. O `-` final faz parte do nome do recurso; o comando tenta usar o ConfigMap `api-config-` e falha. O sufixo `-` só remove variáveis nomeadas (`LOG_LEVEL-`).
>
> Se apagar o ConfigMap com o Deployment ainda a referenciá-lo, os novos pods ficam em `CreateContainerConfigError`.

### 6.7. Secret

**O que faz:** cria um Secret e injeta `API_KEY` no pod da **API**. **Não muda:** imagem, Service, HPA.

```bash
kubectl create secret generic api-secret -n daemon-lab \
  --from-literal=API_KEY=segredo123 \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl set env deployment/api-deployment -n daemon-lab --from=secret/api-secret
kubectl rollout status deployment/api-deployment -n daemon-lab
kubectl exec -it deploy/api-deployment -n daemon-lab -- printenv API_KEY
```

Revert:

```bash
kubectl set env deployment/api-deployment -n daemon-lab API_KEY-
kubectl rollout status deployment/api-deployment -n daemon-lab
kubectl delete secret api-secret -n daemon-lab
```

> O valor aparece em `printenv` de propósito, para a demo. Em produção, evitaria expor segredos no terminal e usaria `valueFrom.secretKeyRef` com o mínimo de privilégios.

### 6.8. Resources (requests/limits)

**O que faz:** lê ou altera **CPU/memória** (requests/limits) da API. **Não muda:** imagem, réplicas, Service.

Inspecionar o que existe:

```bash
kubectl get deployment api-deployment -n daemon-lab \
  -o jsonpath='{.spec.template.spec.containers[0].resources}'; echo
kubectl describe pod -l app=process-api -n daemon-lab | grep -A6 -E "Limits|Requests"
```

A API declara `requests.cpu: 100m` e `limits.cpu: 500m`. O HPA usa o request de CPU como denominador do cálculo de utilização.

Alterar ao vivo:

```bash
kubectl set resources deployment/api-deployment -n daemon-lab \
  --requests=cpu=200m,memory=128Mi \
  --limits=cpu=500m,memory=256Mi
kubectl rollout status deployment/api-deployment -n daemon-lab
```

Revert:

```bash
kubectl apply -f k8s/api-deployment.yaml
kubectl rollout status deployment/api-deployment -n daemon-lab
```

> `requests` afeta o agendamento e o HPA; `limits` afeta o throttling de CPU e o OOMKill de memória. Se `requests == limits` para CPU e memória em todos os containers, o pod passa a QoS `Guaranteed`.

### 6.9. Probes — o que existe (e o que não existe)

**O que faz:** verifica (e, se pedido, adiciona) **probes** ao pod da API. **Não muda:** imagem, réplicas, HPA.

**O projeto não define liveness, readiness nem startup probes.** Este comando devolve vazio:

```bash
kubectl get deployment api-deployment -n daemon-lab \
  -o jsonpath='{.spec.template.spec.containers[0].livenessProbe}'; echo
```

Se o professor pedir para adicionar, use um endpoint leve (`/docs`) e `timeoutSeconds` maior que o padrão de 1 s:

```bash
kubectl patch deployment api-deployment -n daemon-lab --type=strategic -p '
{"spec":{"template":{"spec":{"containers":[{"name":"api",
  "readinessProbe":{"httpGet":{"path":"/docs","port":8000},"initialDelaySeconds":5,"periodSeconds":10,"timeoutSeconds":3},
  "livenessProbe":{"httpGet":{"path":"/docs","port":8000},"initialDelaySeconds":15,"periodSeconds":20,"timeoutSeconds":3}
}]}}}}'
kubectl rollout status deployment/api-deployment -n daemon-lab
kubectl describe pod -l app=process-api -n daemon-lab | grep -A4 -E "Liveness|Readiness"
```

Revert:

```bash
kubectl patch deployment api-deployment -n daemon-lab --type=strategic -p '
{"spec":{"template":{"spec":{"containers":[{"name":"api","livenessProbe":null,"readinessProbe":null}]}}}}'
kubectl rollout status deployment/api-deployment -n daemon-lab
```

> Evite usar `/processes` como probe: ele enumera todos os processos e pode demorar mais de 1 s sob carga, provocando reinícios em cascata. O ideal seria um endpoint dedicado `/health`.

### 6.10. Diagnóstico rápido

**O que faz:** mostra estado, eventos, logs e consumo. **Não muda:** nada; é só leitura.

```bash
kubectl get pods -n daemon-lab -o wide
kubectl describe pod <nome-do-pod> -n daemon-lab
kubectl logs <nome-do-pod> -n daemon-lab
kubectl logs deploy/api-deployment -n daemon-lab --tail=50
kubectl get events -n daemon-lab --sort-by=.lastTimestamp
kubectl top pods -n daemon-lab
kubectl top nodes
```

Erros comuns e onde olhar:

| Sintoma | Causa provável | Verificação |
|---|---|---|
| `ErrImageNeverPull` | imagem não carregada no Minikube | `minikube image ls \| grep process-api` |
| `ImagePullBackOff` | tentou fazer pull de imagem inexistente | `kubectl describe pod` |
| `CreateContainerConfigError` | ConfigMap/Secret referenciado não existe | `kubectl get configmap,secret -n daemon-lab` |
| `Pending` | sem recursos ou nodeSelector impossível | `kubectl describe pod` (eventos) |
| HPA `<unknown>` | metrics-server ausente ou ainda a arrancar | `kubectl top pods -n daemon-lab` |
| `cron.txt` inexistente | primeiro tick ainda não ocorreu | aguardar ~60 s |
| `port-forward` falha | porta local ocupada | usar outra porta local |

---

## 7. Reset entre demos

### 7.1. Reset sem apagar o namespace (preferido)

```bash
# 1) garantir a imagem original
kubectl set image deployment/api-deployment api=process-api:latest -n daemon-lab

# 2) remover variáveis adicionadas (ignore erros se não existirem)
kubectl set env deployment/api-deployment -n daemon-lab MINHA_VAR- LOG_LEVEL- API_KEY- 2>/dev/null || true

# 3) devolver as réplicas
kubectl scale deployment nginx-deployment -n daemon-lab --replicas=2
kubectl scale deployment cron-deployment  -n daemon-lab --replicas=1

# 4) reaplicar os manifestos originais
kubectl apply -f k8s/

# 5) confirmar
kubectl rollout status deployment/api-deployment   -n daemon-lab
kubectl rollout status deployment/nginx-deployment -n daemon-lab
kubectl rollout status deployment/cron-deployment  -n daemon-lab
kubectl get pods,hpa,svc -n daemon-lab
```

> Se durante a apresentação editou ficheiros de `k8s/`, restaure-os antes do `apply`: `git restore k8s/`.
>
> Evite `kubectl rollout undo` como reset genérico: ele volta apenas uma revisão e pode reaplicar `process-api:v2` se houve mais de um rollout.

### 7.2. Reset total do namespace

```bash
kubectl delete namespace daemon-lab --ignore-not-found
kubectl wait --for=delete namespace/daemon-lab --timeout=180s

kubectl apply -f k8s/namespace.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Active namespace/daemon-lab --timeout=60s

kubectl apply -f k8s/api-deployment.yaml
kubectl apply -f k8s/api-service.yaml
kubectl apply -f k8s/nginx-deployment.yaml
kubectl apply -f k8s/nginx-service.yaml
kubectl apply -f k8s/cron-deployment.yaml
kubectl apply -f k8s/hpa.yaml

kubectl wait --for=condition=Available deployment/api-deployment   -n daemon-lab --timeout=180s
kubectl wait --for=condition=Available deployment/nginx-deployment -n daemon-lab --timeout=180s
kubectl wait --for=condition=Available deployment/cron-deployment  -n daemon-lab --timeout=180s
```

> O `kubectl delete namespace` é assíncrono. Sem o `kubectl wait --for=delete`, o `apply` imediato pode falhar com `object is being deleted: namespaces "daemon-lab" already exists`.
>
> As imagens continuam carregadas no Minikube; não é preciso `minikube image load` de novo. O metrics-server é um addon do cluster e também permanece ativo.

---

## 8. Encerramento e recuperação

### 8.1. Pausar (mantém tudo)

```bash
minikube stop
```

Para retomar:

```bash
minikube start --driver=docker
kubectl config use-context minikube
kubectl get pods -n daemon-lab
```

### 8.2. Apagar o cluster (só se necessário)

`minikube delete` remove o cluster, as imagens carregadas e os addons. Se usar, **é preciso reconstruir tudo**:

```bash
minikube start --driver=docker
kubectl config use-context minikube

minikube image load process-api:latest
minikube image load nginx-cron:latest
minikube image load cron-daemon:latest
minikube image load process-api:v2

minikube addons enable metrics-server
kubectl wait --for=condition=Available deployment/metrics-server -n kube-system --timeout=180s

# O namespace tem de existir antes dos recursos que o usam
kubectl apply -f k8s/namespace.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Active namespace/daemon-lab --timeout=60s

kubectl apply -f k8s/api-deployment.yaml
kubectl apply -f k8s/api-service.yaml
kubectl apply -f k8s/nginx-deployment.yaml
kubectl apply -f k8s/nginx-service.yaml
kubectl apply -f k8s/cron-deployment.yaml
kubectl apply -f k8s/hpa.yaml

kubectl wait --for=condition=Available deployment/api-deployment   -n daemon-lab --timeout=180s
kubectl wait --for=condition=Available deployment/nginx-deployment -n daemon-lab --timeout=180s
kubectl wait --for=condition=Available deployment/cron-deployment  -n daemon-lab --timeout=180s
```

> Se as imagens também foram removidas do Docker local (`docker rmi`), refaça a secção 1 antes do `minikube image load`.

---

## 9. Checklist de 60 segundos antes de apresentar

```bash
docker ps | grep -i docker
minikube status
kubectl config current-context
kubectl get nodes
kubectl get pods -n daemon-lab
kubectl get hpa -n daemon-lab
kubectl top pods -n daemon-lab
```

- [ ] Terminal no **Ubuntu/WSL**, na raiz do repositório.
- [ ] 5 pods `Running` e `READY` (2 API + 2 Nginx + 1 cron).
- [ ] `kubectl top pods` responde (metrics-server ativo).
- [ ] `curl` à API devolve JSON.
- [ ] `cron.txt` e `cron-task.log` já têm pelo menos uma linha.
- [ ] Imagem `process-api:v2` existe no Minikube (`minikube image ls | grep v2`).
- [ ] Plano B de acesso pronto (`kubectl port-forward`).
- [ ] Este guia aberto localmente — não depender de rede para consultar.
