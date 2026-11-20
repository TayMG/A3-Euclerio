# Modificações

---

## Parte 1 — Modificações apenas com terminal (sem editar YAML)

### 1.1. Escalar o número de réplicas do Deployment

**Conceito:** um `Deployment` mantém `N` réplicas via `ReplicaSet`. O `kubectl scale` altera `spec.replicas` diretamente.

**Aplicar:**
```bash
kubectl scale deployment api-deployment --replicas=4 -n daemon-lab
```

**Verificar:**
```bash
kubectl get pods -n daemon-lab -w
kubectl get deploy api-deployment -n daemon-lab
```

**Reverter:**
```bash
kubectl scale deployment api-deployment --replicas=2 -n daemon-lab
```

**O que dizer:** o `ReplicaSet` cria pods até ao número desejado; o `Service` distribui tráfego via `Endpoints`. Se o HPA estiver ativo e a CPU baixa, ele reduz novamente para o `minReplicas` — para o scale permanecer, ver 1.3.

**Espelhar para Nginx/Cron** (se o professor pedir outro serviço):
```bash
kubectl scale deployment nginx-deployment --replicas=3 -n daemon-lab
kubectl scale deployment cron-deployment  --replicas=3 -n daemon-lab
```

---

### 1.2. Alterar o alvo de CPU do HPA (ex: 50% → 70%)

**Conceito:** o HPA compara o uso médio de CPU dos pods com `averageUtilization`. Uso > alvo → escala para cima; uso < alvo → escala para baixo.

**Confirmar a versão da API antes:**
```bash
kubectl get hpa api-hpa -n daemon-lab -o yaml | grep "apiVersion"
```

**Aplicar (se `autoscaling/v2`):**
```bash
kubectl patch hpa api-hpa -n daemon-lab --type=json \
  -p='[{"op": "replace", "path": "/spec/metrics/0/resource/target/averageUtilization", "value": 70}]'
```

**Aplicar (se `autoscaling/v1`):**
```bash
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"targetCPUUtilizationPercentage":70}}'
```

**Verificar:**
```bash
kubectl get hpa -n daemon-lab
kubectl describe hpa api-hpa -n daemon-lab | grep -A 3 Metrics
```

**Reverter (v2):**
```bash
kubectl patch hpa api-hpa -n daemon-lab --type=json \
  -p='[{"op": "replace", "path": "/spec/metrics/0/resource/target/averageUtilization", "value": 50}]'
```

**Reverter (v1):**
```bash
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"targetCPUUtilizationPercentage":50}}'
```

**O que dizer:** alvo mais alto (70%) → HPA mais tolerante, só escala acima dos 70%. Alvo mais baixo (30%) → escala mais cedo.

---

### 1.3. Alterar o `minReplicas` do HPA

**Conceito:** piso de réplicas que o HPA mantém mesmo com carga baixa. Útil para forçar o scale manual a “colar”.

**Aplicar:**
```bash
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"minReplicas":4}}'
```

**Verificar:**
```bash
kubectl get hpa -n daemon-lab
kubectl get pods -n daemon-lab -w
```

**Reverter (valor original do projeto: `minReplicas: 2`):**
```bash
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"minReplicas":2}}'
```

**O que dizer:** com `minReplicas=4`, o HPA nunca desce abaixo de 4 pods. Explica porque o `kubectl scale --replicas=4` “fica” — o HPA concorda.

---

### 1.4. Alterar o `maxReplicas` do HPA

**Conceito:** teto de réplicas que o HPA pode criar.

**Aplicar:**
```bash
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"maxReplicas":15}}'
```

**Verificar:**
```bash
kubectl get hpa -n daemon-lab
```

**Reverter (valor original do projeto: `maxReplicas: 10`):**
```bash
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"maxReplicas":10}}'
```

**O que dizer:** mesmo com carga alta, o HPA nunca cria mais de `maxReplicas` pods. Proteção contra picos.

---

### 1.5. Alterar uma variável de ambiente do Deployment

**Conceito:** `kubectl set env` altera o `spec.template` do Deployment, o que dispara um **rolling update** automático.

**Aplicar:**
```bash
kubectl set env deployment/api-deployment MINHA_VAR=novo -n daemon-lab
```

**Verificar:**
```bash
kubectl rollout status deployment/api-deployment -n daemon-lab
kubectl exec -it deploy/api-deployment -n daemon-lab -- env | grep MINHA_VAR
```

**Reverter (o `-` no fim remove a variável):**
```bash
kubectl set env deployment/api-deployment MINHA_VAR- -n daemon-lab
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**O que dizer:** `set env` altera o template do pod, o `ReplicaSet` é recriado e os pods são substituídos um a um (rolling update), sem downtime.

---

### 1.6. Trocar a imagem e fazer rollback (rolling update + undo)

**Conceito:** `kubectl set image` dispara um rolling update. `kubectl rollout undo` volta à revisão anterior usando o histórico do Deployment.

**Pré-requisito — criar a imagem `v2` no Minikube (por causa do `imagePullPolicy: Never`):**
```bash
docker tag process-api:latest process-api:v2
minikube image load process-api:v2
```

**Confirmar o nome do container (no projeto é `api`):**
```bash
kubectl get deploy api-deployment -n daemon-lab \
  -o jsonpath='{.spec.template.spec.containers[*].name}'
```

**Aplicar:**
```bash
kubectl set image deployment/api-deployment api=process-api:v2 -n daemon-lab
```

**Verificar:**
```bash
kubectl rollout status deployment/api-deployment -n daemon-lab
kubectl rollout history deployment/api-deployment -n daemon-lab
kubectl get pods -n daemon-lab -o wide
```

**Reverter:**
```bash
kubectl rollout undo deployment/api-deployment -n daemon-lab
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**O que dizer:** o Deployment guarda `revisionHistoryLimit` revisões. O `undo` volta à anterior sem precisares do YAML original. Se a `v2` não estivesse carregada no Minikube, os pods ficariam em `ErrImageNeverPull` por causa do `imagePullPolicy: Never`.

---

### 1.7. Rolling restart (reiniciar pods sem alterar nada)

**Conceito:** força a recriação dos pods do Deployment — útil após rebuild da imagem `:latest` com `imagePullPolicy: Never`.

**Aplicar:**
```bash
kubectl rollout restart deployment/api-deployment -n daemon-lab
```

**Verificar:**
```bash
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**Reverter:** não aplicável (não é uma alteração de configuração).

**O que dizer:** equivalente a apagar os pods todos, mas de forma controlada (um a um) e sem downtime.

---

### 1.8. Criar um ConfigMap, montar como env e reverter

**Conceito:** ConfigMaps desacoplam configuração do código. `kubectl set env --from=configmap` injeta as chaves como variáveis de ambiente.

**Aplicar:**
```bash
kubectl create configmap api-config -n daemon-lab --from-literal=LOG_LEVEL=debug
kubectl set env deployment/api-deployment -n daemon-lab --from=configmap/api-config
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**Verificar:**
```bash
kubectl get configmap api-config -n daemon-lab -o yaml
kubectl exec -it deploy/api-deployment -n daemon-lab -- env | grep LOG_LEVEL
```

**Reverter:**
```bash
kubectl set env deployment/api-deployment -n daemon-lab --from=configmap/api-config-
kubectl delete configmap api-config -n daemon-lab
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**O que dizer:** o ConfigMap pode ser montado como variável de ambiente ou como volume. Como env, cada alteração requer rollout. Como volume, o ficheiro é atualizado sem reiniciar o pod (mas a app tem de reler).

---

### 1.9. Criar um Secret, montar como env e reverter

**Conceito:** igual ao ConfigMap, mas para dados sensíveis (base64). O Kubernetes não os mostra em `describe`.

**Aplicar:**
```bash
kubectl create secret generic api-secret -n daemon-lab --from-literal=API_KEY=segredo123
kubectl set env deployment/api-deployment -n daemon-lab --from=secret/api-secret
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**Verificar:**
```bash
kubectl get secret api-secret -n daemon-lab
kubectl describe secret api-secret -n daemon-lab
kubectl exec -it deploy/api-deployment -n daemon-lab -- env | grep API_KEY
```

**Reverter:**
```bash
kubectl set env deployment/api-deployment -n daemon-lab --from=secret/api-secret-
kubectl delete secret api-secret -n daemon-lab
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**O que dizer:** o Secret não é encriptado por defeito — apenas base64. Em produção real usar-se-ia um KMS externo (Sealed Secrets, Vault). No escopo do projeto, demonstra a separação config/sensível.

---

### 1.10. Alterar o tipo do Service (NodePort → ClusterIP)

**Conceito:** `ClusterIP` só é acessível dentro do cluster. `NodePort` expõe numa porta fixa do nó. O `api-service` do projeto tem `type: NodePort` e `nodePort: 30080`. Ao mudar para `ClusterIP`, o `nodePort` **tem** de ser removido na mesma operação — senão o Kubernetes rejeita.

**Aplicar (patch JSON atómico: muda o tipo e remove o nodePort):**
```bash
kubectl patch svc api-service -n daemon-lab --type=json \
  -p='[{"op":"replace","path":"/spec/type","value":"ClusterIP"},{"op":"remove","path":"/spec/ports/0/nodePort"}]'
```

**Verificar:**
```bash
kubectl get svc api-service -n daemon-lab
# Deve mostrar TYPE=ClusterIP e sem coluna nodePort preenchida
# Acesso só via port-forward:
kubectl port-forward svc/api-service 8000:8000 -n daemon-lab
# Noutro terminal:
curl http://127.0.0.1:8000/processes
```

**Reverter (repor NodePort 30080 do projeto):**
```bash
kubectl patch svc api-service -n daemon-lab --type=json \
  -p='[{"op":"replace","path":"/spec/type","value":"NodePort"},{"op":"add","path":"/spec/ports/0/nodePort","value":30080}]'
kubectl get svc api-service -n daemon-lab
```

**O que dizer:** `NodePort` foi escolhido nas interfaces para acesso via `minikube service`. `ClusterIP` é o tipo por defeito e o mais comum em produção (Ingress fica à frente). O `nodePort` só é válido quando `type` é `NodePort` ou `LoadBalancer` — daí a necessidade de o remover no mesmo patch.

---

### 1.11. Port-forward como plano B (quando `minikube service` falha)

**Conceito:** `kubectl port-forward` cria um túnel local → pod/serviço, sem depender do `minikube service`.

**Aplicar:**
```bash
kubectl port-forward svc/api-service 8000:8000 -n daemon-lab
# Deixar este terminal aberto
```

**Verificar (noutro terminal):**
```bash
curl http://127.0.0.1:8000/processes
curl http://127.0.0.1:8000/docs
```

**Reverter:** `Ctrl+C`.

**O que dizer:** túnel direto ao pod via API do Kubernetes. Bom para debug e para demos quando a rede do host não permite NodePort.

---

### 1.12. Testar todos os endpoints da API (gestão de processos)

**Conceito:** o tema do projeto é **gestão de processos**. O professor pode pedir `POST` e `DELETE`.

**Aplicar (com `<PORTA>` do `minikube service` ou `8000` do port-forward):**
```bash
# Listar processos
curl http://127.0.0.1:<PORTA>/processes

# Inspecionar um PID específico
curl http://127.0.0.1:<PORTA>/processes/1

# Iniciar um processo (parâmetro command, conforme interfaces)
curl -X POST "http://127.0.0.1:<PORTA>/processes/start?command=sleep%2060"

# Guardar o PID devolvido e encerrá-lo (force opcional, padrão false)
curl -X DELETE "http://127.0.0.1:<PORTA>/processes/<PID>?force=true"

# Documentação Swagger
# Abrir no navegador: http://127.0.0.1:<PORTA>/docs
```

**Verificar no pod (via psutil, com hostPID: true):**
```bash
kubectl exec -it deploy/api-deployment -n daemon-lab -- ps aux | grep sleep
```

**Reverter:** `DELETE` no processo criado (se ficou órfão).

**O que dizer:** a API usa `psutil` e, como o pod tem `hostPID: true`, vê os processos do nó. O `POST` cria com `subprocess.Popen`; o `DELETE` usa `psutil.Process(pid).terminate()` ou `kill()` se `force=true`.

---

### 1.13. Forçar o HPA a reagir (ciclo completo: carga → escala → parar → desce)

**Conceito:** o HPA reage a métricas reais. Para o ver a escalar e a descer **sozinho** (sem `kubectl scale`), gera-se carga na API e depois para-se. É o demo mais convincente para o professor.

**Pré-requisito (já garantido pelo projeto):** `api-deployment.yaml` tem `resources.requests.cpu: 100m`, logo o HPA consegue calcular a percentagem. `metrics-server` ativo:
```bash
kubectl top pods -n daemon-lab
kubectl get hpa -n daemon-lab   # TARGETS deve mostrar %, não <unknown>
```

**Passo 1 — observar o estado inicial (noutro terminal):**
```bash
kubectl get hpa -n daemon-lab -w
kubectl get pods -n daemon-lab -w
```

**Passo 2 — gerar carga:**
```bash
# Opção A (preferida): port-forward + loop de curl no host
kubectl port-forward svc/api-service 8000:8000 -n daemon-lab &
while true; do curl -s http://127.0.0.1:8000/processes > /dev/null; done
# Ctrl+C para parar a carga

# Opção B (plano B — pod de stress com Python, já que a imagem da API não tem wget):
kubectl run stress --image=process-api:latest -n daemon-lab --restart=Never -- \
  python -c "import urllib.request; [urllib.request.urlopen('http://api-service:8000/processes') for _ in iter(int,1)]"
# Limpar depois: kubectl delete pod stress -n daemon-lab
```

> **Porque não `busybox`/`wget`:** a imagem da API é `python:3.11-slim` (sem `wget`) e o `imagePullPolicy: Never` impede puxar `busybox` do Docker Hub. Usar a própria imagem do projeto evita downloads.

**Verificar (o que deves ver):**
- `kubectl get hpa -w` → `TARGETS` sobe acima de 50%, `REPLICAS` aumenta (2 → 3 → 4).
- `kubectl get pods -w` → novos pods `ContainerCreating` → `Running`.
- `kubectl get deploy api-deployment -n daemon-lab` → `READY` aumenta.

**Passo 3 — parar a carga e ver o HPA a descer:**
```bash
# Ctrl+C no loop, ou:
kubectl delete pod stress -n daemon-lab
```

**Verificar (após ~1–2 min):**
- `TARGETS` volta a < 50%.
- `REPLICAS` desce gradualmente até `minReplicas` (4 → 3 → 2).
- Pods a terminar (`Terminating`) e a desaparecer.

**Reverter:** nada a reverter — o HPA volta sozinho ao `minReplicas`. Se ficou pod de stress, apagar.

**O que dizer:** o HPA lê métricas a cada 15s e calcula `replicas = ceil(replicas_atuais * uso_atual / alvo)`. Com carga, o uso sobe acima de 50% e escala para cima; sem carga, desce. Tem um período de estabilização (`behavior.scaleDown.stabilizationWindowSeconds`) que evita flapping — por isso a descida pode demorar 1–2 min. Isto demonstra o **ciclo reativo completo**, não apenas o scale manual.

---

## Parte 2 — Modificações que exigem edição de YAML

> **Formato comum:** editar o ficheiro em `k8s/`, aplicar com `kubectl apply -f`, verificar com `describe`/`logs`. Para reverter: `kubectl rollout undo` ou editar de volta e aplicar.

### 2.1. Adicionar liveness e readiness probes à API

**Conceito:** `readinessProbe` controla se o pod recebe tráfego (falha → removido dos `Endpoints`). `livenessProbe` controla se o pod deve ser reiniciado (falha → `kubelet` mata e recria o contentor).

**Editar `k8s/api-deployment.yaml`**, dentro do `containers[0]`:
```yaml
        readinessProbe:
          httpGet:
            path: /processes
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /processes
            port: 8000
          initialDelaySeconds: 15
          periodSeconds: 10
          failureThreshold: 3
```

**Aplicar:**
```bash
kubectl apply -f k8s/api-deployment.yaml
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**Verificar:**
```bash
kubectl describe pod -l app=process-api -n daemon-lab | grep -A 4 -i "liveness\|readiness"
kubectl get pods -n daemon-lab
```

**Reverter:**
```bash
kubectl rollout undo deployment/api-deployment -n daemon-lab
```

**O que dizer:** sem probes, o Kubernetes considera o pod “pronto” assim que o contentor arranca. Com probes, só recebe tráfego se responder a `/processes`. A `liveness` mata o pod se a app bloquear.

**Espelhar para Nginx** (se pedido):
```yaml
        readinessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 15
          periodSeconds: 10
```

---

### 2.2. Adicionar resource requests e limits

**Conceito:** `requests` é o que o scheduler reserva no nó. `limits` é o teto. CPU excedida → throttling; memória excedida → OOMKill. A razão `requests/limits` define a classe QoS.

> **Nota:** o `api-deployment.yaml` do projeto **já tem** `requests.cpu: 100m` e `limits.cpu: 500m`. Esta secção serve para alterar esses valores ou para adicionar memória.

**Editar `k8s/api-deployment.yaml`**, dentro do `containers[0]`:
```yaml
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
          limits:
            cpu: "800m"
            memory: "256Mi"
```

**Aplicar:**
```bash
kubectl apply -f k8s/api-deployment.yaml
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**Verificar:**
```bash
kubectl describe pod -l app=process-api -n daemon-lab | grep -A 5 -i "limits\|requests"
kubectl get pod -l app=process-api -n daemon-lab -o jsonpath='{.items[0].status.qosClass}'
kubectl top pods -n daemon-lab
```

**Reverter:**
```bash
kubectl rollout undo deployment/api-deployment -n daemon-lab
```

**O que dizer:** sem `requests`, o scheduler pode sobrecarregar um nó. Sem `limits`, um pod pode consumir toda a CPU/memória. Com ambos e `requests==limits`, QoS `Guaranteed`; aqui fica `Burstable`.

---

### 2.3. Adicionar `securityContext` (correr como non-root)

**Conceito:** por defeito, os contentores correm como `root`. `runAsNonRoot` obriga a imagem a ter `USER` não-root. `allowPrivilegeEscalation: false` bloqueia `sudo`/`setuid`. `capabilities.drop: ALL` remove capabilities.

**Editar `k8s/api-deployment.yaml`**, dentro do `containers[0]`:
```yaml
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
```

> **Atenção:** **não** incluir `readOnlyRootFilesystem: true` ao vivo — pode quebrar a app se ela escrever em `/tmp` ou no diretório de trabalho. Se quiseres demonstrar, faz numa cópia e reverte logo.

**Aplicar:**
```bash
kubectl apply -f k8s/api-deployment.yaml
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**Verificar:**
```bash
kubectl describe pod -l app=process-api -n daemon-lab | grep -A 6 -i "securityContext"
kubectl exec -it deploy/api-deployment -n daemon-lab -- id
# Deve mostrar uid=1000, não uid=0
```

**Reverter:**
```bash
kubectl rollout undo deployment/api-deployment -n daemon-lab
```

**O que dizer:** `hostPID: true` foi escolha intencional do projeto (ler processos do nó) e já é uma limitação de segurança a citar no artigo. `securityContext` mitiga outros riscos (root, capabilities).

---

### 2.4. Discutir `hostPID: true` (e porque não removê-lo no escopo)

**Conceito:** `hostPID: true` faz o pod partilhar o namespace de PID do nó. Sem isto, `psutil` só veria os processos do próprio contentor — **inviabiliza o tema do projeto**.

**Verificar o estado atual:**
```bash
kubectl get deploy api-deployment -n daemon-lab -o yaml | grep -A 1 hostPID
# Deve mostrar: hostPID: true
```

**Demonstração do efeito:**
```bash
kubectl exec -it deploy/api-deployment -n daemon-lab -- ps aux | head -20
# Deve listar processos do nó, não só do contentor
```

**O que dizer:** `hostPID: true` é intencional e está documentado como limitação de segurança. Em produção, seria substituído por uma API que acede a `/proc` via `hostPath` restrito ou por um DaemonSet privilegiado. Para o escopo académico, é o que permite demonstrar a gestão de processos. Justificação a constar no artigo.

---

### 2.5. Adicionar PersistentVolumeClaim (persistir dados do Cron)

**Conceito:** `emptyDir` morre com o pod. `PVC` persiste além do ciclo de vida do pod. No Minikube, a `StorageClass` por defeito (`standard`) faz `hostPath` dinamicamente — **sem download nem addon**.

**Editar `k8s/cron-deployment.yaml`**, adicionar em `spec.template.spec`:
```yaml
      volumes:
        - name: cron-data
          persistentVolumeClaim:
            claimName: cron-pvc
```

E no contentor:
```yaml
          volumeMounts:
            - name: cron-data
              mountPath: /data
```

**Criar `k8s/cron-pvc.yaml`:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cron-pvc
  namespace: daemon-lab
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 1Gi
```

**Aplicar:**
```bash
kubectl apply -f k8s/cron-pvc.yaml
kubectl apply -f k8s/cron-deployment.yaml
kubectl rollout status deployment/cron-deployment -n daemon-lab
```

**Verificar:**
```bash
kubectl get pvc -n daemon-lab
kubectl get pv
# Escrever, apagar o pod, verificar persistência
kubectl exec -it deploy/cron-deployment -n daemon-lab -- sh -c "echo teste > /data/persist.txt"
kubectl delete pod -l app=cron -n daemon-lab
kubectl exec -it deploy/cron-deployment -n daemon-lab -- cat /data/persist.txt
```

**Reverter:**
```bash
kubectl delete -f k8s/cron-pvc.yaml
kubectl rollout undo deployment/cron-deployment -n daemon-lab
```

**O que dizer:** sem PVC, o `emptyDir` perde tudo quando o pod morre. Com PVC, o volume é “reclamado” pelo novo pod. `ReadWriteOnce` = um nó de cada vez.

---

### 2.6. Converter o Cron de Deployment para CronJob

**Conceito:** um `Deployment` mantém o pod sempre a correr. Um `CronJob` cria um `Job` em horários definidos por cron. Para um “daemon” que escreve periodicamente, `CronJob` é mais idiomático.

**Criar `k8s/cron-cronjob.yaml`:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cron-daemon
  namespace: daemon-lab
spec:
  schedule: "* * * * *"        # a cada minuto (demo)
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cron
              image: cron-daemon:latest
              imagePullPolicy: Never
              command: ["sh", "-c", "date >> /shared/cron.txt"]
              volumeMounts:
                - name: shared
                  mountPath: /shared
          volumes:
            - name: shared
              persistentVolumeClaim:
                claimName: cron-pvc
```

**Aplicar (remover o Deployment antigo e criar o CronJob):**
```bash
kubectl delete deployment cron-deployment -n daemon-lab
kubectl apply -f k8s/cron-cronjob.yaml
```

**Verificar:**
```bash
kubectl get cronjob -n daemon-lab
kubectl get jobs -n daemon-lab
kubectl get pods -n daemon-lab | grep cron
kubectl logs -l job-name --tail=20
```

**Reverter:**
```bash
kubectl delete cronjob cron-daemon -n daemon-lab
kubectl apply -f k8s/cron-deployment.yaml
```

**O que dizer:** `CronJob` cria `Job`s em horários definidos; cada `Job` cria um pod que termina. Para o escopo do projeto, o `Deployment` demonstra self-healing contínuo (mais visível), mas o `CronJob` é a forma correta para tarefas periódicas. Ambos são válidos.

---

### 2.7. Adicionar `nodeSelector` (controlar onde corre)

**Conceito:** `nodeSelector` fixa o pod a nós com um label. Em clusters multi-nó, controla a distribuição.

**Verificar labels dos nós:**
```bash
kubectl get nodes --show-labels
```

**Editar `k8s/api-deployment.yaml`**, dentro de `spec.template.spec`:
```yaml
      nodeSelector:
        kubernetes.io/os: linux
```

**Aplicar:**
```bash
kubectl apply -f k8s/api-deployment.yaml
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**Verificar:**
```bash
kubectl get pods -n daemon-lab -o wide
kubectl describe pod -l app=process-api -n daemon-lab | grep -A 2 "Node-Selectors"
```

**Reverter:**
```bash
kubectl rollout undo deployment/api-deployment -n daemon-lab
```

**O que dizer:** no Minikube (single-node) o efeito prático é nulo, mas o conceito é o mesmo: em clusters multi-nó, `nodeSelector` distribui pods por zonas/tipos de hardware.

---

## Parte 3 — Como editar YAML com segurança

### 3.1. Fluxo recomendado

1. Editar o ficheiro em `k8s/` no editor.
2. Aplicar com `kubectl apply -f k8s/<ficheiro>.yaml`.
3. Verificar com `kubectl rollout status` + `kubectl describe pod`.
4. Se quebrar: `kubectl rollout undo deployment/<nome> -n daemon-lab`.

### 3.2. Validação antes de aplicar

```bash
# Dry-run: valida o YAML sem aplicar
kubectl apply -f k8s/<ficheiro>.yaml --dry-run=server

# Diff: mostra o que vai mudar
kubectl diff -f k8s/<ficheiro>.yaml
```

### 3.3. Edição rápida sem sair do terminal

```bash
kubectl edit deployment/api-deployment -n daemon-lab
```

> **Atenção:** `kubectl edit` altera o recurso **no cluster**, mas **não** o ficheiro em `k8s/`. Se depois fizeres `kubectl apply -f`, o ficheiro sobrescreve a edição.

### 3.4. Reverter uma alteração YAML

```bash
# Voltar à revisão anterior (Deployments)
kubectl rollout undo deployment/<nome> -n daemon-lab

# Ver histórico
kubectl rollout history deployment/<nome> -n daemon-lab

# Voltar a uma revisão específica
kubectl rollout undo deployment/<nome> -n daemon-lab --to-revision=1

# Para recursos que não são Deployment (Service, ConfigMap, etc.):
kubectl delete -f k8s/<ficheiro>.yaml
kubectl apply -f k8s/<ficheiro>.yaml
```

---

## Parte 5 — Reset total das modificações

Se o professor pedir muitas alterações e o ambiente ficar inconsistente:

```bash
# Apagar tudo e redeployar do zero
kubectl delete namespace daemon-lab

# Recriar
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/api-deployment.yaml
kubectl apply -f k8s/api-service.yaml
kubectl apply -f k8s/nginx-deployment.yaml
kubectl apply -f k8s/nginx-service.yaml
kubectl apply -f k8s/cron-deployment.yaml
kubectl apply -f k8s/hpa.yaml

# Confirmar
kubectl get all -n daemon-lab
```
