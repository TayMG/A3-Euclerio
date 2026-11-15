# Projeto A3 — Tema 1

**Orquestração de Serviços Daemon com Kubernetes e API de Gerenciamento de Processos**

## 📝 Descrição

Ambiente distribuído contendo 3 containers orquestrados através do Kubernetes no Minikube:
1. **API FastAPI:** Gerenciamento e inspeção de processos do sistema hospedeiro (via `psutil` e `hostPID`).
2. **Nginx com Cron:** Servidor web rodando em conjunto com um daemon cron embutido no mesmo container.
3. **Daemon Cron Isolado:** Container rodando exclusivamente tarefas agendadas em background.

O projeto demonstra na prática conceitos de Sistemas Operacionais, containerização e orquestração, incluindo validação de resiliência (*Self-healing*) e escalabilidade automática (*Horizontal Pod Autoscaler - HPA*).

---

## 🛠️ Tecnologias Utilizadas

- **Linguagens e Frameworks:** Python 3.11, FastAPI, Uvicorn, `psutil`
- **Servidores e Daemons:** Nginx, Cron (BusyBox/Alpine)
- **Containerização:** Docker
- **Orquestração:** Kubernetes (Minikube, `kubectl`)

---

## 📁 Estrutura do Repositório

```text
projeto-a3/
├── api/                    # (Tarefa 1) API em Python
│   ├── main.py
│   ├── requirements.txt
│   └── Dockerfile
├── nginx/                  # (Tarefa 2) Web server + Cron
│   ├── html/
│   │   └── index.html
│   └── Dockerfile
├── cron/                   # (Tarefa 2) Daemon isolado
│   └── Dockerfile
├── k8s/                    # (Tarefa 3) Manifestos do Kubernetes
│   ├── namespace.yaml
│   ├── api-deployment.yaml
│   ├── api-service.yaml
│   ├── nginx-deployment.yaml
│   ├── nginx-service.yaml
│   ├── cron-deployment.yaml
│   └── hpa.yaml
├── docs/                   # (Tarefa 4) Documentação e evidências
│   ├── interfaces.md
│   ├── tarefas.md
│   ├── artigo.md           
│   ├── slides.pdf          
│   └── evidencias/         
├── .gitignore
└── README.md               
```

---

## 🚀 Como Executar

Siga os passos abaixo para rodar o projeto localmente. Certifique-se de ter o Docker e o Minikube instalados.

### 1. Build das imagens Docker

Execute os comandos abaixo na raiz do repositório para compilar as imagens com as tags exatas configuradas nos manifestos:

```bash
cd api && docker build -t process-api:latest .
cd ../nginx && docker build -t nginx-cron:latest .
cd ../cron && docker build -t cron-daemon:latest .
cd ..
```

### 2. Iniciar e carregar imagens no Minikube

Inicie o cluster e injete as imagens compiladas diretamente no Minikube:

```bash
minikube start --driver=docker
minikube image load process-api:latest
minikube image load nginx-cron:latest
minikube image load cron-daemon:latest
```

### 3. Aplicar os YAMLs do Kubernetes

Faça o deploy de todos os recursos no namespace `daemon-lab`:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/api-deployment.yaml
kubectl apply -f k8s/api-service.yaml
kubectl apply -f k8s/nginx-deployment.yaml
kubectl apply -f k8s/nginx-service.yaml
kubectl apply -f k8s/cron-deployment.yaml
kubectl apply -f k8s/hpa.yaml
```

Verifique se tudo está rodando corretamente:
```bash
kubectl get pods -n daemon-lab
kubectl get svc -n daemon-lab
kubectl get hpa -n daemon-lab
```

### 4. Como acessar a API

A API está exposta via `NodePort` na porta `30080`. Você pode acessá-la de duas formas:

```bash
curl http://$(minikube ip):30080/processes
```
Ou criando um túnel automático pelo Minikube:
```bash
minikube service api-service -n daemon-lab
```

---

## 🔌 Endpoints da API

A interface interativa do Swagger pode ser acessada adicionando `/docs` à URL base da API.

| Método | Rota | Descrição |
|--------|------|-----------|
| `GET` | `/processes` | Lista todos os processos ativos no nó. |
| `GET` | `/processes/{pid}` | Inspeciona dados detalhados de um processo pelo PID. |
| `POST` | `/processes/start` | Inicia um novo processo no sistema (via `command`). |
| `DELETE` | `/processes/{pid}` | Encerra um processo pelo PID (aceita flag `force`). |

---

## 📄 Documentação Completa

- [Regras e Interfaces do Projeto](docs/interfaces.md)
- [Divisão de Tarefas](docs/tarefas.md)
- [Artigo Técnico (Completo)](docs/artigo.md)
- [Apresentação em Slides](docs/slides.pdf)
