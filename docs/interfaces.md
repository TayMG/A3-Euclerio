# Documento de Interfaces do Projeto

**Tema 1: Orquestração de Serviços Daemon com Kubernetes e API de Gerenciamento de Processos**

---

## 1. Nomes das Imagens Docker

As imagens serão construídas com estes nomes **exatos**. Ninguém pode mudar sem avisar o grupo.

| Imagem | Nome completo | Quem constrói | Dockerfile |
|--------|---------------|---------------|------------|
| API | `process-api:latest` | Pessoa 1 | `api/Dockerfile` |
| Nginx + Cron | `nginx-cron:latest` | Pessoa 2 | `nginx/Dockerfile` |
| Cron separado | `cron-daemon:latest` | Pessoa 2 | `cron/Dockerfile` |

**Regra:** todos os YAMLs do Kubernetes devem usar exatamente esses nomes, com `imagePullPolicy: Never`.

---

## 2. Portas

### Portas dos containers

| Serviço | Porta interna |
|---------|---------------|
| API | 8000 |
| Nginx | 80 |
| Cron | (não expõe porta) |

### Portas dos Services no Kubernetes

| Service | Tipo | port | targetPort | nodePort |
|---------|------|------|------------|----------|
| api-service | NodePort | 8000 | 8000 | 30080 |
| nginx-service | NodePort | 80 | 80 | 30081 |

### Como acessar a API

```bash
http://$(minikube ip):30080/processes
# ou
minikube service api-service -n daemon-lab
```

**Regra:** ninguém muda as portas sem avisar o grupo.

---

## 3. Labels (Etiquetas dos Pods)

| Recurso | Label |
|---------|-------|
| API | `app: process-api` |
| Nginx | `app: nginx` |
| Cron | `app: cron` |

**Regra:** o label do Deployment **tem** que ser igual ao selector do Service. Se um usar `app: api` e o outro `app: process-api`, o Service não encontra os Pods.

---

## 4. Namespace

**Namespace do projeto:** `daemon-lab`

Todos os recursos (Deployments, Services, HPA) devem estar neste namespace.

```bash
kubectl get pods -n daemon-lab
kubectl get svc -n daemon-lab
kubectl get hpa -n daemon-lab
```

**Regra:** ninguém usa o namespace `default` para este projeto.

---

## 5. Endpoints da API

A API FastAPI vai expor **exatamente** estes endpoints:

| Método | Rota | O que faz |
|--------|------|-----------|
| GET | `/processes` | Lista todos os processos |
| GET | `/processes/{pid}` | Inspeciona um processo pelo PID |
| POST | `/processes/start` | Inicia um novo processo |
| DELETE | `/processes/{pid}` | Encerra um processo pelo PID |

### Parâmetros

- `{pid}`: número inteiro (`int`)
- `command` (no POST): string com o comando a executar
- `force` (no DELETE): booleano (opcional, padrão `false`)

### Exemplos de uso

```bash
curl http://127.0.0.1:8000/processes
curl http://127.0.0.1:8000/processes/1
curl -X POST "http://127.0.0.1:8000/processes/start?command=sleep 60"
curl -X DELETE "http://127.0.0.1:8000/processes/123"
```

### Documentação automática

```
http://127.0.0.1:8000/docs
```

**Regra:** ninguém muda o nome das rotas sem avisar o grupo.

---

## 6. Estrutura de Pastas do Repositório

```
projeto-a3/
├── api/                    (Pessoa 1)
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── nginx/                  (Pessoa 2)
│   ├── Dockerfile
│   └── entrypoint.sh
├── cron/                   (Pessoa 2)
│   ├── Dockerfile
│   └── crontab
├── k8s/                    (Pessoa 3)
│   ├── namespace.yaml
│   ├── api-deployment.yaml
│   ├── api-service.yaml
│   ├── nginx-deployment.yaml
│   ├── cron-deployment.yaml
│   └── hpa.yaml
├── docs/                   (Pessoa 4)
│   ├── interfaces.md
│   ├── tarefas.md
│   ├── guia-pessoa-1.md
│   ├── guia-pessoa-2.md
│   ├── guia-pessoa-3.md
│   ├── guia-pessoa-4.md
│   ├── artigo.md
│   ├── slides.pdf
│   └── evidencias/
│       ├── pessoa-1/
│       ├── pessoa-2/
│       ├── pessoa-3/
│       └── pessoa-4/
├── scripts/                (todos)
│   ├── build.sh
│   └── deploy.sh
└── README.md               (Pessoa 4)
```

**Regra:** cada pessoa só mexe na **sua** pasta. Se precisar mexer na pasta de outro, avisar antes.

---

## 7. Branches do GitHub

### Branch principal

- `main` — sempre estável, só recebe Pull Requests aprovados.

### Branches de trabalho

| Branch | Responsável |
|--------|-------------|
| `feature/api` | Pessoa 1 |
| `feature/containers` | Pessoa 2 |
| `feature/k8s` | Pessoa 3 |
| `docs/artigo` | Pessoa 4 |

### Fluxo

1. Cada um trabalha na sua branch.
2. Ao terminar, abre Pull Request para a `main`.
3. Outro integrante revisa e faz merge.
4. Uma vez por semana, todos fazem `git pull` da `main`.
5. **Nunca** commitar direto na `main`.

### Comandos básicos

```bash
git checkout -b feature/api     # criar branch
git add .                        # adicionar arquivos
git commit -m "mensagem"         # commitar
git push origin feature/api      # enviar para GitHub
git pull origin main             # atualizar da main
```

---

## 8. Comandos Padronizados

### Build das imagens

```bash
cd api && docker build -t process-api:latest .
cd nginx && docker build -t nginx-cron:latest .
cd cron && docker build -t cron-daemon:latest .
```

### Carregar imagens no Minikube

```bash
minikube image load process-api:latest
minikube image load nginx-cron:latest
minikube image load cron-daemon:latest
```

### Aplicar YAMLs

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/api-deployment.yaml
kubectl apply -f k8s/api-service.yaml
kubectl apply -f k8s/nginx-deployment.yaml
kubectl apply -f k8s/cron-deployment.yaml
kubectl apply -f k8s/hpa.yaml
```

### Verificar

```bash
kubectl get pods -n daemon-lab
kubectl get svc -n daemon-lab
kubectl get hpa -n daemon-lab
```

### Deletar tudo e refazer

```bash
kubectl delete namespace daemon-lab
minikube delete
minikube start --driver=docker
```

---

## 9. Responsabilidades por Pessoa

> A divisão de tarefas está detalhada em `docs/tarefas.md`.
> Cada pessoa deve seguir a sua guia (`docs/guia-pessoa-X.md`).

### Pessoa 1 — API FastAPI + psutil

- **Guia completa:** `docs/guia-pessoa-1.md`
- **Entregáveis:** `api/main.py`, `api/requirements.txt`, `api/Dockerfile`, prints da API funcionando
- **Branch:** `feature/api`

### Pessoa 2 — Containers Nginx e Cron

- **Guia completa:** `docs/guia-pessoa-2.md`
- **Entregáveis:** `nginx/Dockerfile`, `nginx/entrypoint.sh`, `cron/Dockerfile`, prints dos containers rodando
- **Branch:** `feature/containers`

### Pessoa 3 — Kubernetes (YAMLs + deploy + self-healing + scaling)

- **Guia completa:** `docs/guia-pessoa-3.md`
- **Entregáveis:** todos os `k8s/*.yaml`, prints do `kubectl get pods/svc/hpa`, vídeo do self-healing e scaling
- **Branch:** `feature/k8s`

### Pessoa 4 — Documentação + Integração + Demonstração

- **Guia completa:** `docs/guia-pessoa-4.md`
- **Entregáveis:** `docs/artigo.md`, `docs/slides.pdf`, `README.md`, roteiro de apresentação
- **Branch:** `docs/artigo`

---

## 11. Regras de Ouro

1. **Cada um na sua branch.** Ninguém mexe na branch do outro.
2. **Pull Request sempre.** Nada vai direto para a `main`.
3. **Integração semanal.** Toda semana, uma reunião rápida para alinhar.
4. **Documentar tudo.** Prints, vídeos, anotações. Vira o artigo no final.
5. **Avisar se mudar algo.** Se precisar mudar uma interface, avisar **todo** o grupo antes.
