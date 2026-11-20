# Modificações

> **Como usar este guia:** cada secção tem **Conceito → Aplicar → Verificar → Reverter → O que dizer**. Execute tudo no terminal do **Ubuntu/WSL**, na raiz do repositório. Os nomes de imagens, namespace, portas, labels e endpoints seguem `docs/interfaces.md`.
>
> **Regra de ouro:** nada aqui deve exigir download durante a apresentação. As imagens do projeto já estão no Minikube e o `metrics-server` deve estar ativo antes da demo.
>
> **Reverter YAML:** se editou ficheiros de `k8s/`, faça `git restore k8s/` antes de reaplicar. `kubectl rollout undo` reverte o Deployment no cluster, mas não o ficheiro local.
>
> **Leitura rápida:** cada secção começa com **Muda / Não muda**. `minReplicas` mexe só no **piso**; `maxReplicas` só no **teto**; `averageUtilization` só no **alvo de CPU**; `set env`, `set image` e `set resources` mexem na **API**; `scale nginx` mexe só no **Nginx**; `patch svc` mexe só na **exposição**.

---

## Parte 1 — Modificações apenas com terminal (sem editar YAML)

### 1.1. Escalar o número de réplicas do Deployment

**Muda:** o número de pods do **Nginx**. **Não muda:** API, HPA, imagens, portas.

**Conceito:** um `Deployment` mantém `N` réplicas via `ReplicaSet`. O `kubectl scale` altera `spec.replicas` diretamente.

**Aplicar (Nginx, que não é gerido pelo HPA):**
```bash
kubectl scale deployment nginx-deployment --replicas=3 -n daemon-lab
```

**Verificar:**
```bash
kubectl get pods -l app=nginx -n daemon-lab -w
kubectl get deploy nginx-deployment -n daemon-lab
```

**Reverter:**
```bash
kubectl scale deployment nginx-deployment --replicas=2 -n daemon-lab
```

**O que dizer:** o `ReplicaSet` cria pods até ao número desejado; o `Service` distribui tráfego via `Endpoints`.

> **Cuidado com a API:** `api-deployment` é gerido pelo HPA. `kubectl scale deployment api-deployment --replicas=4` funciona por instantes, mas o HPA volta ao valor calculado (normalmente `minReplicas: 2`). Para manter 4 réplicas, altere o `minReplicas` (secção 1.3).
>
> **Cuidado com o cron:** escalar `cron-deployment` para 3 réplicas cria **três daemons independentes**, cada um a executar a mesma tarefa e a escrever no seu próprio log. Sem volume compartilhado ou leader election, isso duplica trabalho. Prefira demonstrar escala no Nginx.

---

### 1.2. Alterar o alvo de CPU do HPA (ex: 50% → 70%)

**Muda:** o **alvo de CPU** do HPA da API. **Não muda:** o número de pods imediatamente, a imagem, o Service.

**Conceito:** o HPA compara o uso médio de CPU dos pods com `averageUtilization`. Uso > alvo → escala para cima; uso < alvo → escala para baixo.

**Confirmar a versão da API antes:**
```bash
kubectl get hpa api-hpa -n daemon-lab -o jsonpath='{.apiVersion}{"\n"}'
```

**Aplicar (se `autoscaling/v2`):**
```bash
# Muda o ALVO de CPU do HPA da API (50% -> 70%); não cria pods sozinho
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

**O que dizer:** alvo mais alto (70%) → HPA mais tolerante, só escala acima dos 70%. Alvo mais baixo (30%) → escala mais cedo. O projeto usa `autoscaling/v2`; o caminho v1 existe apenas para comparação.

---

### 1.3. Alterar o `minReplicas` do HPA

**Muda:** o **mínimo** de réplicas do HPA (piso de pods da API). **Não muda:** o máximo, a imagem, o Service.

**Conceito:** piso de réplicas que o HPA mantém mesmo com carga baixa. Útil para forçar o scale manual a “colar”.

**Aplicar:**
```bash
# Muda o MÍNIMO de pods da API (piso = 4); não muda o máximo
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"minReplicas":4}}'
```

**Verificar:**
```bash
kubectl get hpa -n daemon-lab
kubectl get pods -l app=process-api -n daemon-lab -w
```

**Reverter (valor original do projeto: `minReplicas: 2`):**
```bash
kubectl patch hpa api-hpa -n daemon-lab -p '{"spec":{"minReplicas":2}}'
```

**O que dizer:** com `minReplicas=4`, o HPA nunca desce abaixo de 4 pods. Explica porque o `kubectl scale --replicas=4` “fica” — o HPA concorda. Com `minReplicas=2`, o HPA pode reduzir até 2.

---

### 1.4. Alterar o `maxReplicas` do HPA

**Muda:** o **máximo** de réplicas do HPA (teto de pods da API). **Não muda:** o mínimo, a imagem, o Service.

**Conceito:** teto de réplicas que o HPA pode criar.

**Aplicar:**
```bash
# Muda o MÁXIMO de pods da API (teto = 15); não muda o mínimo
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

**O que dizer:** mesmo com carga alta, o HPA nunca cria mais de `maxReplicas` pods. Proteção contra picos e contra custo descontrolado.

> `maxReplicas` tem de ser maior ou igual a `minReplicas`; o API server rejeita o patch caso contrário.

---

### 1.5. Alterar uma variável de ambiente do Deployment

**Muda:** uma variável de ambiente no pod da **API** (dispara novo rollout). **Não muda:** imagem, Service, HPA.

**Conceito:** `kubectl set env` altera o `spec.template` do Deployment, o que dispara um **rolling update** automático.

**Aplicar:**
```bash
# Adiciona MINHA_VAR ao pod da API e reinicia os pods (rolling update)
kubectl set env deployment/api-deployment MINHA_VAR=novo -n daemon-lab
```

**Verificar:**
```bash
kubectl rollout status deployment/api-deployment -n daemon-lab
kubectl exec -it deploy/api-deployment -n daemon-lab -- printenv MINHA_VAR
```

**Reverter (o `-` no fim remove a variável):**
```bash
kubectl set env deployment/api-deployment MINHA_VAR- -n daemon-lab
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**O que dizer:** `set env` altera o template do pod, o `ReplicaSet` é recriado e os pods são substituídos um a um (rolling update), sem downtime.

> `printenv` é mais direto do que `env | grep` e devolve código de saída 1 se a variável não existir — útil para confirmar a remoção.

---

### 1.6. Trocar a imagem e fazer rollback (rolling update + undo)

**Muda:** a **imagem** do contentor `api` (dispara novo rollout). **Não muda:** configuração, Service, HPA.

**Conceito:** `kubectl set image` dispara um rolling update. `kubectl rollout undo` volta à revisão anterior usando o histórico do Deployment.

**Pré-requisito — criar a imagem `v2` no Minikube (por causa do `imagePullPolicy: Never`):**
```bash
docker tag process-api:latest process-api:v2
minikube image load process-api:v2
```

**Confirmar o nome do container (no projeto é `api`):**
```bash
kubectl get deploy api-deployment -n daemon-lab \
  -o jsonpath='{.spec.template.spec.containers[*].name}{"\n"}'
```

**Aplicar:**
```bash
# Troca só a imagem do contentor 'api'; não mexe em env, Service nem HPA
kubectl set image deployment/api-deployment api=process-api:v2 -n daemon-lab
```

**Verificar:**
```bash
kubectl rollout status deployment/api-deployment -n daemon-lab
kubectl rollout history deployment/api-deployment -n daemon-lab
kubectl get pods -l app=process-api -n daemon-lab -o wide
```

**Reverter:**
```bash
kubectl rollout undo deployment/api-deployment -n daemon-lab
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**O que dizer:** o Deployment guarda `revisionHistoryLimit` revisões. O `undo` volta à anterior sem precisar do YAML original. Se a `v2` não estivesse carregada no Minikube, os pods ficariam em `ErrImageNeverPull` por causa do `imagePullPolicy: Never`.

> `rollout undo` volta **uma** revisão. Se fizer vários ciclos, confirme a imagem final e, se necessário, force o estado original:
>
> ```bash
> kubectl set image deployment/api-deployment api=process-api:latest -n daemon-lab
> kubectl rollout status deployment/api-deployment -n daemon-lab
> ```

---

### 1.7. Rolling restart (reiniciar pods sem alterar nada)

**Muda:** recria os pods da **API** com a mesma configuração. **Não muda:** imagem, env, Service.

**Conceito:** força a recriação dos pods do Deployment — útil após rebuild da imagem `:latest` com `imagePullPolicy: Never`.

**Aplicar:**
```bash
kubectl rollout restart deployment/api-deployment -n daemon-lab
```

**Verificar:**
```bash
kubectl rollout status deployment/api-deployment -n daemon-lab
kubectl get pods -l app=process-api -n daemon-lab
```

**Reverter:** não aplicável (não é uma alteração de configuração).

**O que dizer:** equivalente a apagar os pods todos, mas de forma controlada (um a um) e sem downtime. Como a tag `:latest` não muda, o restart é a forma de fazer o kubelet usar a imagem recém-carregada.

---

### 1.8. Criar um ConfigMap, montar como env e reverter

**Muda:** adiciona `LOG_LEVEL` ao pod da **API**, vindo de um ConfigMap. **Não muda:** imagem, Service, HPA.

**Conceito:** ConfigMaps desacoplam configuração do código. `kubectl set env --from=configmap` injeta as chaves como variáveis de ambiente.

**Aplicar (idempotente):**
```bash
kubectl create configmap api-config -n daemon-lab \
  --from-literal=LOG_LEVEL=debug \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl set env deployment/api-deployment -n daemon-lab --from=configmap/api-config
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**Verificar:**
```bash
kubectl get configmap api-config -n daemon-lab -o yaml
kubectl exec -it deploy/api-deployment -n daemon-lab -- printenv LOG_LEVEL
```

**Reverter (a ordem importa):**
```bash
# 1) remover a referência do pod template
kubectl set env deployment/api-deployment -n daemon-lab LOG_LEVEL-
kubectl rollout status deployment/api-deployment -n daemon-lab

# 2) só agora apagar o ConfigMap
kubectl delete configmap api-config -n daemon-lab
```

**O que dizer:** o ConfigMap pode ser montado como variável de ambiente ou como volume. Como env, cada alteração requer rollout. Como volume, o ficheiro é atualizado sem reiniciar o pod (mas a app tem de reler).

> **Erro clássico:** `kubectl set env ... --from=configmap/api-config-` **não remove** nada. O `-` final faz parte do nome do recurso; o comando tenta usar o ConfigMap `api-config-` e falha. O sufixo `-` só remove variáveis nomeadas (`LOG_LEVEL-`).
>
> Se apagar o ConfigMap com o Deployment ainda a referenciá-lo, os novos pods ficam em `CreateContainerConfigError`.

---

### 1.9. Criar um Secret, montar como env e reverter

**Muda:** adiciona `API_KEY` ao pod da **API**, vindo de um Secret. **Não muda:** imagem, Service, HPA.

**Conceito:** igual ao ConfigMap, mas para dados sensíveis (base64). O Kubernetes não os mostra em `describe`.

**Aplicar (idempotente):**
```bash
kubectl create secret generic api-secret -n daemon-lab \
  --from-literal=API_KEY=segredo123 \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl set env deployment/api-deployment -n daemon-lab --from=secret/api-secret
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**Verificar:**
```bash
kubectl get secret api-secret -n daemon-lab
kubectl describe secret api-secret -n daemon-lab
kubectl exec -it deploy/api-deployment -n daemon-lab -- printenv API_KEY
```

**Reverter:**
```bash
kubectl set env deployment/api-deployment -n daemon-lab API_KEY-
kubectl rollout status deployment/api-deployment -n daemon-lab
kubectl delete secret api-secret -n daemon-lab
```

**O que dizer:** o Secret não é encriptado por defeito — apenas base64. Em produção real usar-se-ia um KMS externo (Sealed Secrets, Vault, External Secrets). No escopo do projeto, demonstra a separação entre configuração e dados sensíveis.

> O valor aparece em `printenv` de propósito, para a demo. Em produção, evitaria expor segredos no terminal e usaria `valueFrom.secretKeyRef` com o mínimo de privilégios.

---

### 1.10. Alterar o tipo do Service (NodePort → ClusterIP)

**Muda:** o tipo de exposição do **Service da API**. **Não muda:** pods, porta interna 8000, imagem.

**Conceito:** `ClusterIP` só é acessível dentro do cluster. `NodePort` expõe numa porta fixa do nó. O `api-service` do projeto tem `type: NodePort` e `nodePort: 30080`. Ao mudar para `ClusterIP`, o `nodePort` **tem** de ser removido na mesma operação — senão o Kubernetes rejeita.

**Aplicar (patch JSON atómico: muda o tipo e remove o nodePort):**
```bash
# Muda o Service da API para ClusterIP e remove a porta externa
kubectl patch svc api-service -n daemon-lab --type=json \
  -p='[{"op":"replace","path":"/spec/type","value":"ClusterIP"},{"op":"remove","path":"/spec/ports/0/nodePort"}]'
```

**Verificar:**
```bash
kubectl get svc api-service -n daemon-lab
# Deve mostrar TYPE=ClusterIP e sem nodePort preenchido
# Acesso só via port-forward:
kubectl port-forward svc/api-service 8000:8000 -n daemon-lab
# Noutro terminal:
curl -s http://127.0.0.1:8000/processes | head -c 500
```

**Reverter (repor NodePort 30080 do projeto):**
```bash
kubectl patch svc api-service -n daemon-lab --type=json \
  -p='[{"op":"replace","path":"/spec/type","value":"NodePort"},{"op":"add","path":"/spec/ports/0/nodePort","value":30080}]'
kubectl get svc api-service -n daemon-lab
```

**O que dizer:** `NodePort` foi escolhido nas interfaces para acesso via `minikube service`. `ClusterIP` é o tipo por defeito e o mais comum em produção (Ingress fica à frente). O `nodePort` só é válido quando `type` é `NodePort` ou `LoadBalancer` — daí a necessidade de o remover no mesmo patch.

> Se a porta 30080 estiver ocupada por outro Serviço, o patch de revert falha. Nesse caso, use outra porta (`30090`) e ajuste a demonstração.

---

### 1.11. Port-forward como plano B (quando `minikube service` falha)

**Muda:** nada no cluster; cria um **túnel local** para o Service. **Não muda:** Service, pods, HPA.

**Conceito:** `kubectl port-forward` cria um túnel local → pod/serviço, sem depender do `minikube service`.

**Aplicar:**
```bash
kubectl port-forward svc/api-service 8000:8000 -n daemon-lab
# Deixar este terminal aberto
```

**Verificar (noutro terminal):**
```bash
curl -s http://127.0.0.1:8000/processes | head -c 500
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8000/docs
```

**Reverter:** `Ctrl+C`.

**O que dizer:** túnel direto ao pod via API do Kubernetes. Bom para debug e para demos quando a rede do host não permite NodePort. Se a porta 8000 já estiver ocupada, use `8001:8000` e ajuste os `curl`.

---

### 1.12. Testar todos os endpoints da API (gestão de processos)

**Muda:** nada de configuração; **cria e encerra processos** através da API. **Não muda:** Deployment, Service, HPA.

**Conceito:** o tema do projeto é **gestão de processos**. O professor pode pedir `POST` e `DELETE`.

**Aplicar (com `<PORTA>` do `minikube service` ou `8000` do port-forward):**
```bash
# Listar processos
curl -s http://127.0.0.1:<PORTA>/processes | head -c 1500

# Inspecionar um PID específico
curl -s http://127.0.0.1:<PORTA>/processes/1

# Iniciar um processo (use 600s para dar tempo de copiar o PID e encerrar)
curl -X POST "http://127.0.0.1:<PORTA>/processes/start?command=sleep%20600"

# Guardar o PID devolvido e encerrá-lo (force opcional, padrão false)
curl -X DELETE "http://127.0.0.1:<PORTA>/processes/<PID>?force=true"

# Documentação Swagger
# Abrir no navegador: http://127.0.0.1:<PORTA>/docs
```

**Verificar no pod (a imagem não tem `ps`):**
```bash
# A API expõe os processos; o mais fiável é consultar o próprio endpoint
curl -s http://127.0.0.1:<PORTA>/processes | grep -i sleep

# Alternativa dentro do pod: listar /proc (não requer procps)
kubectl exec -it deploy/api-deployment -n daemon-lab -- sh -c 'ls /proc | grep -E "^[0-9]+$" | head'
```

**Reverter:** `DELETE` no processo criado (se ficou órfão).

**O que dizer:** a API usa `psutil` e, como o pod tem `hostPID: true`, vê os processos do nó. O `POST` cria com `subprocess.Popen`; o `DELETE` usa `psutil.Process(pid).terminate()` ou `kill()` se `force=true`.

> **Não use `ps aux`:** a imagem base é `python:3.11-slim` e não instala `procps`. O comando falha com `executable file not found in $PATH`. Use o endpoint `/processes` ou `ls /proc`.
>
> **Não use `sleep 60`:** o processo pode terminar antes de copiar o PID e chamar `DELETE`. Use `sleep 600` e limpe no fim.

---

### 1.13. Forçar o HPA a reagir (ciclo completo: carga → escala → parar → desce)

**Muda:** nada de configuração; **gera carga** e observa o HPA. **Não muda:** alvo, mínimo e máximo (salvo o patch opcional de `scaleDown`).

**Conceito:** o HPA reage a métricas reais. Para o ver a escalar e a descer **sozinho** (sem `kubectl scale`), gera-se carga na API e depois para-se.

**Pré-requisito (já garantido pelo projeto):** `api-deployment.yaml` tem `resources.requests.cpu: 100m`, logo o HPA consegue calcular a percentagem. `metrics-server` ativo:
```bash
kubectl top pods -n daemon-lab
kubectl get hpa -n daemon-lab   # TARGETS deve mostrar %, não <unknown>
```

**Passo 1 — observar o estado inicial (noutro terminal):**
```bash
kubectl get hpa api-hpa -n daemon-lab -w
kubectl get pods -l app=process-api -n daemon-lab -w
```

**Passo 2 — gerar carga (Opção A: loop de `curl` no host):**
```bash
kubectl port-forward svc/api-service 8000:8000 -n daemon-lab &
while true; do curl -s http://127.0.0.1:8000/processes > /dev/null; done
# Ctrl+C para parar a carga; depois: kill %1 (encerra o port-forward)
```

**Opção B (pod de stress com a imagem do projeto, sem downloads):**
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
# Limpar depois: kubectl delete pod stress -n daemon-lab
```

> **Por que `--image-pull-policy=Never`:** com a tag `:latest`, o servidor assume `Always`. Sem a flag, o pod fica em `ErrImagePull`/`ErrImagePullBackOff` mesmo com a imagem presente no nó. `Never` garante o uso da imagem local.
>
> **Por que não `busybox`:** `busybox` não está no Minikube e seria baixado do Docker Hub; além disso, `imagePullPolicy: Never` nos manifestos do projeto **não** se aplica a um pod criado com `kubectl run`. Usar a imagem do projeto evita downloads e mantém a demo offline.
>
> **Por que não list comprehension:** `[urlopen(...) for _ in iter(int, 1)]` acumula todas as respostas na memória até o pod ser OOMKilled. O `while True` com `.read()` mantém o consumo estável.

**Verificar (o que deve ver):**
- `kubectl get hpa -w` → `TARGETS` sobe acima de 50%, `REPLICAS` aumenta (2 → 3 → 4).
- `kubectl get pods -w` → novos pods `ContainerCreating` → `Running`.
- `kubectl get deploy api-deployment -n daemon-lab` → `READY` aumenta.

**Passo 3 — parar a carga:**
```bash
# Ctrl+C no loop de curl; ou, se usou o pod:
kubectl delete pod stress -n daemon-lab
# Se o port-forward ficou em background:
kill %1 2>/dev/null || true
```

**Verificar (após ~5 min):**
- `TARGETS` volta a < 50%.
- `REPLICAS` desce gradualmente até `minReplicas` (4 → 3 → 2).
- Pods a terminar (`Terminating`) e a desaparecer.

**Reverter:** nada a reverter — o HPA volta sozinho ao `minReplicas`. Se ficou pod de stress, apagar.

**O que dizer:** o HPA lê métricas a cada 15 s e calcula `replicas = ceil(replicas_atuais * uso_atual / alvo)`. Com carga, o uso sobe acima de 50% e escala para cima; sem carga, desce. O período de estabilização padrão (`behavior.scaleDown.stabilizationWindowSeconds: 300`) evita flapping — por isso a descida demora **cerca de 5 minutos**, não 1–2.

> Para uma demo mais rápida, reduza a janela **antes** de gerar carga:
>
> ```bash
> kubectl patch hpa api-hpa -n daemon-lab --type=merge \
>   -p '{"spec":{"behavior":{"scaleDown":{"stabilizationWindowSeconds":60}}}}'
> ```
>
> Reverta no fim com `kubectl apply -f k8s/hpa.yaml` (ou recrie o HPA, se o campo persistir).

---

## Parte 2 — Modificações que exigem edição de YAML

> **Formato comum:** editar o ficheiro em `k8s/`, aplicar com `kubectl apply -f`, verificar com `describe`/`logs`. Para reverter: `git restore k8s/<ficheiro>.yaml`, aplicar de novo, ou `kubectl rollout undo`.

### 2.1. Adicionar liveness e readiness probes à API

**Muda:** adiciona **probes** ao pod da API (dispara novo rollout). **Não muda:** imagem, réplicas, HPA.

**Conceito:** `readinessProbe` controla se o pod recebe tráfego (falha → removido dos `Endpoints`). `livenessProbe` controla se o pod deve ser reiniciado (falha → `kubelet` mata e recria o contentor).

**Editar `k8s/api-deployment.yaml`**, dentro do `containers[0]`:
```yaml
        readinessProbe:
          httpGet:
            path: /docs
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
        livenessProbe:
          httpGet:
            path: /docs
            port: 8000
          initialDelaySeconds: 15
          periodSeconds: 20
          timeoutSeconds: 3
          failureThreshold: 3
```

**Aplicar:**
```bash
kubectl apply -f k8s/api-deployment.yaml
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**Verificar:**
```bash
kubectl describe pod -l app=process-api -n daemon-lab | grep -A 5 -i "liveness\|readiness"
kubectl get pods -l app=process-api -n daemon-lab
```

**Reverter:**
```bash
git restore k8s/api-deployment.yaml
kubectl apply -f k8s/api-deployment.yaml
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**O que dizer:** sem probes, o Kubernetes considera o pod “pronto” assim que o contentor arranca. Com probes, só recebe tráfego se responder. A `liveness` mata o pod se a app bloquear.

> **Não use `/processes` como probe:** o endpoint enumera todos os processos e pode demorar mais de 1 s sob carga. O `timeoutSeconds` padrão é 1 s; isso provocaria reinícios em cascata. Use `/docs` ou, idealmente, um endpoint dedicado `/health` (Parte 4).
>
> O `rollout undo` também reverte as probes, mas não altera o ficheiro local. Prefira `git restore` + `apply` para manter o repositório e o cluster coerentes.

**Espelhar para Nginx** (se pedido):
```yaml
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
          timeoutSeconds: 3
```

---

### 2.2. Adicionar resource requests e limits

**Muda:** **CPU/memória** (requests/limits) da API. **Não muda:** imagem, réplicas, Service.

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
kubectl describe pod -l app=process-api -n daemon-lab | grep -A 6 -i "limits\|requests"
kubectl get pod -l app=process-api -n daemon-lab -o jsonpath='{.items[0].status.qosClass}{"\n"}'
kubectl top pods -n daemon-lab
```

**Reverter:**
```bash
git restore k8s/api-deployment.yaml
kubectl apply -f k8s/api-deployment.yaml
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**O que dizer:** sem `requests`, o scheduler pode sobrecarregar um nó. Sem `limits`, um pod pode consumir toda a CPU/memória. Com ambos e `requests==limits`, QoS `Guaranteed`; aqui fica `Burstable`.

> O HPA usa o `requests.cpu` como denominador. Se remover o request, o HPA passa a mostrar `<unknown>` para CPU.

---

### 2.3. Adicionar `securityContext` (endurecer o contentor)

**Muda:** as **permissões do processo** dentro do pod (capabilities e escalada de privilégios). **Não muda:** imagem, réplicas, HPA.

**Conceito:** por defeito, os contentores correm como `root`. `allowPrivilegeEscalation: false` bloqueia `sudo`/`setuid`. `capabilities.drop: ALL` remove capabilities. `runAsNonRoot`/`runAsUser` obrigam a um utilizador não-root — mas isso exige que a imagem e os diretórios de trabalho permitam esse UID.

**Editar `k8s/api-deployment.yaml`**, dentro do `containers[0]`:
```yaml
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
```

**Aplicar:**
```bash
kubectl apply -f k8s/api-deployment.yaml
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**Verificar:**
```bash
# Mostra o securityContext aplicado e confirma que o contentor arrancou
kubectl describe pod -l app=process-api -n daemon-lab | grep -A 6 -i "securityContext"
kubectl get pods -l app=process-api -n daemon-lab
```

**Reverter:**
```bash
git restore k8s/api-deployment.yaml
kubectl apply -f k8s/api-deployment.yaml
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**O que dizer:** retirar capabilities e bloquear escalada reduz a superfície de ataque sem quebrar a app: a API ouve na porta 8000 (>1024) e continua a conseguir encerrar processos do mesmo UID.

> **Não aplique `runAsUser: 1000` na API durante a demo principal.** A API usa `psutil` para encerrar processos do nó (via `hostPID: true`). Processos do host pertencem a `root`; um processo a correr como UID 1000 não os consegue terminar — o `DELETE` falha com `PermissionError`/403. A demo de gestão de processos quebraria.
>
> **Não aplique `runAsUser: 101` no Nginx sem preparar a imagem.** O Nginx da imagem oficial abre a porta 80 (<1024) e escreve em `/var/cache/nginx` e `/var/run`; como non-root, falha sem `NET_BIND_SERVICE` e sem diretórios graváveis. Para uma demo real de non-root, use a imagem `nginx-unprivileged` (exigiria download) ou prepare o Dockerfile.
>
> Se quiser **mostrar a falha** como argumento de segurança: aplique `runAsUser: 1000` na API, teste um `DELETE` e reverta em seguida. É uma boa discussão, mas não deixe assim para a demo principal.

---

### 2.4. Discutir `hostPID: true` (e porque não removê-lo no escopo)

**Muda:** nada. Esta secção só **verifica** e **explica**. Não altera a API nem as réplicas.

**Conceito:** `hostPID: true` faz o pod partilhar o namespace de PID do nó. Sem isto, `psutil` só veria os processos do próprio contentor — **inviabiliza o tema do projeto**.

**Verificar o estado atual:**
```bash
# Mostra o campo hostPID do Deployment da API; deve ser true
kubectl get deploy api-deployment -n daemon-lab -o yaml | grep -A 1 hostPID
```

**Demonstração do efeito:**
```bash
# Lista os processos vistos pela API (inclui processos do nó, não só do contentor)
curl -s http://127.0.0.1:<PORTA>/processes | head -c 1500

# Alternativa sem `ps`: conta os PIDs visíveis em /proc
kubectl exec -it deploy/api-deployment -n daemon-lab -- sh -c 'ls /proc | grep -E "^[0-9]+$" | wc -l'
```

**O que dizer:** `hostPID: true` é intencional e está documentado como limitação de segurança. Em produção, seria substituído por uma API que acede a `/proc` via `hostPath` restrito ou por um DaemonSet privilegiado. Para o escopo académico, é o que permite demonstrar a gestão de processos. Justificação a constar no artigo.

> **Não use `ps aux`:** a imagem `python:3.11-slim` não tem `procps`. Use o endpoint `/processes` ou `ls /proc`.

---

### 2.5. Adicionar PersistentVolumeClaim (persistir dados)

**Muda:** adiciona um volume persistente ao **cron-deployment**. Não muda a API nem o HPA.

> **Importante:** o cron grava o log real em `/var/log/cron-task.log`. Montar o PVC em `/data` **não** persiste esse log; persiste apenas o que for escrito em `/data`. Esta secção demonstra o conceito de PVC com um ficheiro genérico. Para persistir o log do cron, veja a nota no fim.

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

**Criar `k8s/cron-pvc.yaml`:** (pede 1 GiB ao StorageClass padrão)
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
# Cria o pedido de volume persistente
kubectl apply -f k8s/cron-pvc.yaml

# Recria o pod do cron com o volume montado
kubectl apply -f k8s/cron-deployment.yaml
kubectl rollout status deployment/cron-deployment -n daemon-lab
```

**Verificar:**
```bash
# PVC deve ficar Bound; PV mostra o volume criado
kubectl get pvc -n daemon-lab
kubectl get pv

# Escreve um ficheiro em /data
kubectl exec -it deploy/cron-deployment -n daemon-lab -- sh -c "echo teste > /data/persist.txt"

# Apaga o pod; o Deployment recria-o
kubectl delete pod -l app=cron -n daemon-lab
kubectl rollout status deployment/cron-deployment -n daemon-lab

# O ficheiro deve continuar lá (persistiu)
kubectl exec -it deploy/cron-deployment -n daemon-lab -- cat /data/persist.txt
```

**Reverter (primeiro o Deployment, depois o PVC):**
```bash
# 1) remove o volume do pod (volta ao estado sem PVC)
git restore k8s/cron-deployment.yaml
kubectl apply -f k8s/cron-deployment.yaml
kubectl rollout status deployment/cron-deployment -n daemon-lab

# 2) só agora apaga o PVC
kubectl delete -f k8s/cron-pvc.yaml
```

**O que dizer:** sem PVC, o `emptyDir` perde tudo quando o pod morre. Com PVC, o volume é “reclamado” pelo novo pod. `ReadWriteOnce` = um nó de cada vez.

> **Para persistir o log real do cron:** o crontab da imagem escreve em `/var/log/cron-task.log`. Para usar o PVC, altere a linha do crontab para `/data/cron-task.log` e recrie a imagem (Parte 4). Alternativa rápida para a demo:
>
> ```bash
> # Substitui o crontab dentro do pod para escrever no volume
> kubectl exec -it deploy/cron-deployment -n daemon-lab -- sh -c \
>   "echo '* * * * * date >> /data/cron-task.log' > /etc/crontabs/root"
> # Reinicia o pod para o crond reler o crontab (volta à imagem original)
> kubectl rollout restart deployment/cron-deployment -n daemon-lab
> ```
>
> O restart repõe o crontab da imagem; para uma alteração permanente, edite `cron/crontab` e reconstrua.

---

### 2.6. Converter o Cron de Deployment para CronJob

**Muda:** substitui o `cron-deployment` por um `CronJob`. Não toca na API nem no HPA. **Depende do `cron-pvc` da secção 2.5.**

**Conceito:** um `Deployment` mantém o pod sempre a correr. Um `CronJob` cria um `Job` em horários definidos por cron. Para um “daemon” que escreve periodicamente, `CronJob` é mais idiomático.

**Criar `k8s/cron-cronjob.yaml`:** (o `tee` faz o `date` aparecer também em `kubectl logs`)
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
              command: ["sh", "-c", "date | tee -a /shared/cron.txt"]
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
# Garante que o PVC existe (secção 2.5); sem ele o pod fica Pending
kubectl get pvc cron-pvc -n daemon-lab

# Remove o Deployment que mantinha o cron sempre a correr
kubectl delete deployment cron-deployment -n daemon-lab

# Cria o CronJob (um Job por minuto)
kubectl apply -f k8s/cron-cronjob.yaml
```

**Verificar:**
```bash
# Lista o CronJob e os Jobs criados
kubectl get cronjob -n daemon-lab
kubectl get jobs -n daemon-lab

# Lista o pod do Job mais recente
kubectl get pods -n daemon-lab | grep cron

# Mostra a saída do Job mais recente (o `tee` imprime o date)
kubectl logs job/$(kubectl get jobs -n daemon-lab \
  --sort-by=.metadata.creationTimestamp \
  -o jsonpath='{.items[-1].metadata.name}')

# Confirma que o PVC continua montado no Job
kubectl get cronjob cron-daemon -n daemon-lab -o jsonpath='{.spec.jobTemplate.spec.template.spec.volumes}{"\n"}'
```

**Reverter:**
```bash
# Remove o CronJob (e os Jobs/Pods associados)
kubectl delete cronjob cron-daemon -n daemon-lab

# Recria o Deployment original do cron
kubectl apply -f k8s/cron-deployment.yaml
kubectl rollout status deployment/cron-deployment -n daemon-lab
```

**O que dizer:** `CronJob` cria `Job`s em horários definidos; cada `Job` cria um pod que termina. Para o escopo do projeto, o `Deployment` demonstra self-healing contínuo (mais visível), mas o `CronJob` é a forma correta para tarefas periódicas. Ambos são válidos.

> **Logs:** se o comando do contentor redirecionar a saída para um ficheiro (`date >> /shared/cron.txt`), `kubectl logs` fica vazio. Use `tee` (como acima) ou leia o ficheiro com `kubectl exec`.

---

### 2.7. Adicionar `nodeSelector` (controlar onde corre)

**Muda:** adiciona uma restrição de agendamento ao **api-deployment**. Não muda réplicas, imagem nem portas.

**Conceito:** `nodeSelector` fixa o pod a nós com um label. Em clusters multi-nó, controla a distribuição.

**Verificar labels dos nós:**
```bash
# Lista os labels de cada nó (o Minikube tem kubernetes.io/os=linux)
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
# Mostra em que nó cada pod está
kubectl get pods -n daemon-lab -o wide

# Mostra o nodeSelector aplicado
kubectl describe pod -l app=process-api -n daemon-lab | grep -A 2 "Node-Selectors"
```

**Reverter:**
```bash
git restore k8s/api-deployment.yaml
kubectl apply -f k8s/api-deployment.yaml
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**O que dizer:** no Minikube (single-node) o efeito prático é nulo, mas o conceito é o mesmo: em clusters multi-nó, `nodeSelector` distribui pods por zonas/tipos de hardware. Se o label não existir em nenhum nó, o pod fica `Pending`.

---

## Parte 3 — Como editar YAML com segurança

### 3.1. Fluxo recomendado

1. Editar o ficheiro em `k8s/` no editor.
2. Validar sem aplicar (ver 3.2).
3. Aplicar com `kubectl apply -f k8s/<ficheiro>.yaml`.
4. Verificar com `kubectl rollout status` + `kubectl describe pod`.
5. Se quebrar: `git restore` + `apply` ou `kubectl rollout undo deployment/<nome> -n daemon-lab`.

### 3.2. Validação antes de aplicar

```bash
# Valida o YAML no servidor, sem aplicar
kubectl apply -f k8s/<ficheiro>.yaml --dry-run=server

# Mostra a diferença entre o ficheiro e o que está no cluster
kubectl diff -f k8s/<ficheiro>.yaml
```

### 3.3. Edição rápida sem sair do terminal

```bash
# Abre o Deployment no editor; ao guardar, o cluster é atualizado
kubectl edit deployment/api-deployment -n daemon-lab
```

> **Atenção:** `kubectl edit` altera o recurso **no cluster**, mas **não** o ficheiro em `k8s/`. Se depois fizer `kubectl apply -f`, o ficheiro sobrescreve a edição.

### 3.4. Reverter uma alteração YAML

```bash
# Volta à revisão anterior do Deployment
kubectl rollout undo deployment/<nome> -n daemon-lab

# Mostra o histórico de revisões
kubectl rollout history deployment/<nome> -n daemon-lab

# Volta a uma revisão específica
kubectl rollout undo deployment/<nome> -n daemon-lab --to-revision=1

# Para recursos que não são Deployment (Service, ConfigMap, etc.):
kubectl delete -f k8s/<ficheiro>.yaml
kubectl apply -f k8s/<ficheiro>.yaml
```

> `rollout undo` não altera o ficheiro local. Para o cluster e o repositório ficarem iguais, use `git restore k8s/<ficheiro>.yaml` e `kubectl apply -f`.

---

## Parte 4 — Alterações de código e imagem (o que muda onde)

> Esta parte cobre o que o professor costuma pedir e que **não** se resolve só com `kubectl patch`: mudar a API, o schedule do cron ou a página do Nginx. O ciclo é sempre **editar → build → carregar no Minikube → reiniciar**.

### 4.1. Adicionar um endpoint `/health` à API

**Muda:** o código da API (`api/main.py`). Não muda réplicas, HPA nem portas. Depois do rebuild, os pods da API são substituídos.

**Editar `api/main.py`** e acrescentar (junto aos outros `@app.get`, depois de `app = FastAPI()`):
```python
@app.get("/health")
def health():
    return {"status": "ok"}
```

**Construir e carregar a nova imagem:**
```bash
# Reconstrói a imagem da API com a tag que os manifestos usam
cd api
docker build -t process-api:latest .
cd ..

# Coloca a imagem atualizada dentro do Minikube
minikube image load process-api:latest

# Reinicia os pods da API para usarem a imagem recém-carregada
kubectl rollout restart deployment/api-deployment -n daemon-lab
kubectl rollout status deployment/api-deployment -n daemon-lab
```

**Verificar:**
```bash
# Deve devolver {"status":"ok"}
curl -s http://127.0.0.1:<PORTA>/health

# Confirma que os pods estão a usar a imagem local
kubectl get pods -l app=process-api -n daemon-lab \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

**Reverter:**
```bash
# Remove o endpoint do ficheiro e repete o ciclo de build/load/restart
git restore api/main.py
cd api && docker build -t process-api:latest . && cd ..
minikube image load process-api:latest
kubectl rollout restart deployment/api-deployment -n daemon-lab
```

**O que dizer:** a imagem é local e o `imagePullPolicy: Never` impede downloads. O `rollout restart` é obrigatório porque a tag `:latest` não muda — sem ele, os pods continuam com a imagem antiga.

> Se o endpoint for adicionado **depois** das probes da secção 2.1, pode trocar `/docs` por `/health`. É mais leve e mais correto semanticamente.

---

### 4.2. Mudar o schedule do cron

**Muda:** o horário da tarefa periódica. Não muda a API nem o HPA. O cron do Nginx e o cron isolado têm ficheiros diferentes.

**Onde está:** o schedule é gravado na imagem, dentro de `/etc/crontabs/root`, no `Dockerfile`:
- `cron/Dockerfile`, linha 3 → cron isolado (`/var/log/cron-task.log`).
- `nginx/Dockerfile`, linha 8 → cron do Nginx (`/usr/share/nginx/html/cron.txt`).

**Editar, por exemplo, para a cada 2 minutos:**
```cron
*/2 * * * * date >> /var/log/cron-task.log
```

**Reconstruir e aplicar:**
```bash
# Reconstrói a imagem do cron isolado
cd cron
docker build -t cron-daemon:latest .
cd ..

# Carrega a imagem no Minikube
minikube image load cron-daemon:latest

# Reinicia o Deployment do cron
kubectl rollout restart deployment/cron-deployment -n daemon-lab
kubectl rollout status deployment/cron-deployment -n daemon-lab
```

**Para o cron do Nginx:**
```bash
cd nginx
docker build -t nginx-cron:latest .
cd ..
minikube image load nginx-cron:latest
kubectl rollout restart deployment/nginx-deployment -n daemon-lab
kubectl rollout status deployment/nginx-deployment -n daemon-lab
```

**Verificar:**
```bash
# O ficheiro deve ganhar uma linha nova no intervalo configurado
kubectl exec -it deploy/cron-deployment -n daemon-lab -- tail -n 5 /var/log/cron-task.log
```

**Reverter:** `git restore cron/Dockerfile nginx/Dockerfile`, reconstruir e recarregar.

**O que dizer:** o crontab está **dentro da imagem**; não basta editar o ficheiro no pod (o restart repõe a versão da imagem). Alterações permanentes exigem rebuild + `minikube image load` + `rollout restart`.

> **Demo rápida sem rebuild:** altere o crontab dentro do pod e espere o próximo minuto:
>
> ```bash
> kubectl exec -it deploy/cron-deployment -n daemon-lab -- sh -c \
>   "echo '*/2 * * * * date >> /var/log/cron-task.log' > /etc/crontabs/root"
> ```
>
> O `crond` relê `/etc/crontabs`; se não apanhar a alteração, `kubectl rollout restart deployment/cron-deployment -n daemon-lab` (e volta à imagem original).

---

### 4.3. Mudar a página do Nginx ao vivo

**Muda:** o `index.html` de **um pod** do Nginx. Não muda a API, o HPA nem o Deployment. Como há 2 réplicas, o navegador pode mostrar o pod que não foi alterado.

**Aplicar num pod específico:**
```bash
# Escolhe um pod do Nginx
kubectl get pods -l app=nginx -n daemon-lab

# Substitui a página desse pod
kubectl exec -it <nome-do-pod-nginx> -n daemon-lab -- sh -c \
  'echo "<h1>Alterado ao vivo</h1>" > /usr/share/nginx/html/index.html'

# Confirma o conteúdo
kubectl exec -it <nome-do-pod-nginx> -n daemon-lab -- cat /usr/share/nginx/html/index.html
```

**Aplicar nos dois pods (se quiser consistência):**
```bash
# Executa o mesmo comando em todos os pods com label app=nginx
for pod in $(kubectl get pods -l app=nginx -n daemon-lab -o name); do
  kubectl exec "$pod" -n daemon-lab -- sh -c \
    'echo "<h1>Alterado ao vivo</h1>" > /usr/share/nginx/html/index.html'
done
```

**Reverter:**
```bash
# Recria os pods a partir da imagem original (o ficheiro do pod é efémero)
kubectl rollout restart deployment/nginx-deployment -n daemon-lab
kubectl rollout status deployment/nginx-deployment -n daemon-lab
```

**O que dizer:** a alteração vive apenas no sistema de ficheiros do pod. Um restart/recriação repõe a página da imagem. Para uma alteração permanente, edite `nginx/html/index.html` e reconstrua a imagem (`docker build -t nginx-cron:latest .` + `minikube image load`).

---

### 4.4. Mudar o número do NodePort (30080 → 30090)

**Muda:** a porta exposta pelo `api-service` no nó. Não muda a porta interna da API (`8000`), as réplicas nem a imagem.

**Aplicar:**
```bash
# Troca o nodePort do Serviço da API para 30090
kubectl patch svc api-service -n daemon-lab --type=json \
  -p='[{"op":"replace","path":"/spec/ports/0/nodePort","value":30090}]'

# Confirma a nova porta
kubectl get svc api-service -n daemon-lab
```

**Verificar:**
```bash
# Abre o túnel na nova porta (o minikube service usa o nodePort atual)
minikube service api-service -n daemon-lab --url

# Ou, diretamente:
curl -s http://$(minikube ip):30090/processes | head -c 500
```

**Reverter:**
```bash
# Volta à porta original do projeto
kubectl patch svc api-service -n daemon-lab --type=json \
  -p='[{"op":"replace","path":"/spec/ports/0/nodePort","value":30080}]'
```

**O que dizer:** `nodePort` é a porta fixa no nó (30000–32767). `port` é a porta do Service e `targetPort` é a porta do contentor. Mudar `nodePort` não reinicia pods; o Serviço é atualizado na hora.

> No driver Docker do Windows, `curl http://$(minikube ip):30090` pode não funcionar. Prefira `minikube service api-service -n daemon-lab` ou `kubectl port-forward`.

---

### 4.5. Criar um HPA para o Nginx (requer CPU request)

**Muda:** adiciona autoescala ao **Nginx**. Hoje só a API tem HPA. O Nginx **não tem** `resources.requests.cpu`, então o HPA não consegue calcular a percentagem.

**Passo 1 — definir CPU request no Nginx:**
```bash
# Reserva 50m de CPU por pod do Nginx (necessário para o HPA)
kubectl set resources deployment/nginx-deployment -n daemon-lab --requests=cpu=50m
kubectl rollout status deployment/nginx-deployment -n daemon-lab
```

**Passo 2 — criar o HPA:**
```bash
# Cria um HPA v2 para o Nginx: 2 a 5 pods, alvo de 60% de CPU
kubectl autoscale deployment nginx-deployment -n daemon-lab \
  --min=2 --max=5 --cpu-percent=60

# Confirma
kubectl get hpa -n daemon-lab
```

**Reverter:**
```bash
# Remove o HPA do Nginx
kubectl delete hpa nginx-deployment -n daemon-lab

# Volta ao manifesto original (sem request de CPU)
kubectl apply -f k8s/nginx-deployment.yaml
kubectl rollout status deployment/nginx-deployment -n daemon-lab
```

**O que dizer:** o HPA precisa de `requests.cpu` em **todos** os contentores do pod. Sem isso, o `TARGETS` fica `<unknown>`. A API já tem `100m`; o Nginx e o cron não tinham — por isso só a API tinha HPA.

> `kubectl autoscale` cria `autoscaling/v2` em versões recentes. Se o cluster devolver v1, o comportamento é equivalente para CPU.

---

## Parte 5 — Reset total das modificações

Se o professor pedir muitas alterações e o ambiente ficar inconsistente:

```bash
# 1) Restaura os YAMLs locais (se foram editados nas secções 2.x)
git restore k8s/

# 1b) Remove ficheiros de demo criados por este guia (se existirem)
rm -f k8s/cron-pvc.yaml k8s/cron-cronjob.yaml

# 2) Apaga o namespace (assíncrono)
kubectl delete namespace daemon-lab --ignore-not-found

# 3) Espera a eliminação terminar antes de recriar
kubectl wait --for=delete namespace/daemon-lab --timeout=180s

# 4) Recria o namespace
kubectl apply -f k8s/namespace.yaml
kubectl wait --for=jsonpath='{.status.phase}'=Active namespace/daemon-lab --timeout=60s

# 5) Reaplica todos os manifestos
kubectl apply -f k8s/api-deployment.yaml
kubectl apply -f k8s/api-service.yaml
kubectl apply -f k8s/nginx-deployment.yaml
kubectl apply -f k8s/nginx-service.yaml
kubectl apply -f k8s/cron-deployment.yaml
kubectl apply -f k8s/hpa.yaml

# 6) Confirma que tudo voltou
kubectl get all -n daemon-lab
kubectl get hpa -n daemon-lab
kubectl rollout status deployment/api-deployment   -n daemon-lab
kubectl rollout status deployment/nginx-deployment -n daemon-lab
kubectl rollout status deployment/cron-deployment  -n daemon-lab
```

**O que faz cada passo:**
- `git restore k8s/` → devolve os ficheiros locais ao estado do repositório.
- `kubectl delete namespace` → apaga **tudo** dentro do namespace (Deployments, Services, HPA, ConfigMaps, Secrets, PVCs).
- `kubectl wait --for=delete` → espera o namespace desaparecer; sem isto, o `apply` seguinte pode falhar com `object is being deleted`.
- `kubectl apply -f k8s/...` → recria os recursos a partir dos manifestos originais.
- `kubectl rollout status` → confirma que os pods ficaram prontos.

> As imagens continuam carregadas no Minikube e o `metrics-server` continua ativo; não é preciso `minikube image load` nem `minikube addons enable` de novo. Se apagou o cluster com `minikube delete`, siga a recuperação em `docs/comandos.md`, secção 8.2.

---

## Parte 6 — Referência rápida: “o que este comando muda?”

| Comando | Muda o quê? | Não muda |
|---|---|---|
| `kubectl scale deployment nginx-deployment --replicas=3` | Nº de pods do **Nginx** | API, HPA, imagens |
| `kubectl scale deployment api-deployment --replicas=4` | Nº de pods da **API** (o HPA pode desfazer) | Imagem, Service |
| `kubectl patch hpa ... minReplicas` | **Piso** de pods da API | Máximo, imagem |
| `kubectl patch hpa ... maxReplicas` | **Teto** de pods da API | Mínimo, imagem |
| `kubectl patch hpa ... averageUtilization` | **Alvo de CPU** do HPA | Nº de pods imediato |
| `kubectl set env deployment/api-deployment ...` | Variável de ambiente da **API** (novo rollout) | Service, HPA |
| `kubectl set image deployment/api-deployment api=...` | **Imagem** do contentor `api` (novo rollout) | Configuração, Service |
| `kubectl rollout restart deployment/api-deployment` | Recria os pods da API com a mesma config | Imagem, Service |
| `kubectl set env ... --from=configmap/api-config` | Injeta chaves do ConfigMap como env da API | ConfigMap em si |
| `kubectl set env ... LOG_LEVEL-` | **Remove** a variável `LOG_LEVEL` da API | ConfigMap (apagar à parte) |
| `kubectl patch svc api-service ... type=ClusterIP` | Tipo de exposição do **Service da API** | Pods, imagem |
| `kubectl patch svc api-service ... nodePort=30090` | Porta externa do **Service da API** | Porta interna 8000 |
| `kubectl patch deployment ... probes` | Probes do pod (reinicia) | Imagem, HPA |
| `kubectl set resources deployment/api-deployment ...` | CPU/memória (requests/limits) | Imagem, réplicas |
| `kubectl apply -f k8s/cron-pvc.yaml` | Cria um **PVC** para o cron | API, HPA |
| `kubectl delete deployment cron-deployment` | Remove o cron em modo Deployment | API, Nginx |
| `kubectl apply -f k8s/cron-cronjob.yaml` | Cria o cron em modo **CronJob** | API, Nginx |
| `kubectl delete namespace daemon-lab` | **Tudo** no namespace | Cluster, imagens, addons |

> **Regra mental:** `scale` mexe em pods; `patch hpa` mexe na **política** de escala; `set env`/`set image`/`set resources` mexem no **pod template** e disparam rollout; `patch svc` mexe só na **exposição**; `rollout undo/restart` mexe na **revisão**, não no ficheiro local.
