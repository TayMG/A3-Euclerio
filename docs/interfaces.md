=====================================================================
DOCUMENTO DE INTERFACES DO PROJETO
Tema 1: Orquestração de Serviços Daemon com Kubernetes e API de
Gerenciamento de Processos
=====================================================================

=====================================================================
1. NOMES DAS IMAGENS DOCKER
=====================================================================

As imagens serão construídas com estes nomes EXATOS. Ninguém pode
mudar sem avisar o grupo.

IMAGEM DA API:
  Nome: process-api
  Tag:  latest
  Nome completo: process-api:latest
  Quem constrói: Pessoa 1
  Onde fica o Dockerfile: api/Dockerfile

IMAGEM DO NGINX (com cron embutido):
  Nome: nginx-cron
  Tag:  latest
  Nome completo: nginx-cron:latest
  Quem constrói: Pessoa 2
  Onde fica o Dockerfile: nginx/Dockerfile

IMAGEM DO CRON (separado):
  Nome: cron-daemon
  Tag:  latest
  Nome completo: cron-daemon:latest
  Quem constrói: Pessoa 2
  Onde fica o Dockerfile: cron/Dockerfile

REGRA:
- Todos os YAMLs do Kubernetes devem usar exatamente esses nomes.
- imagePullPolicy: Never (para usar imagens locais do Minikube).

=====================================================================
2. PORTAS
=====================================================================

PORTAS DOS CONTAINERS:
  API:   8000 (porta interna do container)
  Nginx: 80   (porta interna do container)
  Cron:  não expõe porta (só executa tarefas)

PORTAS DOS SERVICES NO KUBERNETES:
  api-service:
    type: NodePort
    port: 8000
    targetPort: 8000
    nodePort: 30080

  nginx-service (se necessário):
    type: NodePort
    port: 80
    targetPort: 80
    nodePort: 30081

COMO ACESSAR A API:
  http://$(minikube ip):30080/processes
  ou
  minikube service api-service -n daemon-lab

REGRA:
- Ninguém muda as portas sem avisar o grupo.
- Se precisar mudar, todos os YAMLs e a documentação devem ser
  atualizados juntos.

=====================================================================
3. LABELS (ETIQUETAS DOS PODS)
=====================================================================

LABELS DOS DEPLOYMENTS:
  API:   app: process-api
  Nginx: app: nginx
  Cron:  app: cron

LABELS DOS SELECTORS DOS SERVICES:
  api-service:   selector: app: process-api
  nginx-service: selector: app: nginx

REGRA:
- O label do Deployment TEM que ser igual ao selector do Service.
- Se o label for "app: process-api" no Deployment, o Service
  precisa ter "selector: app: process-api".
- Se um usar "app: api" e o outro "app: process-api", o Service
  NÃO encontra os Pods.

=====================================================================
4. NAMESPACE
=====================================================================

NAMESPACE DO PROJETO:
  daemon-lab

TODOS os recursos (Deployments, Services, HPA) devem estar neste
namespace.

COMANDOS:
  kubectl get pods -n daemon-lab
  kubectl get svc -n daemon-lab
  kubectl get hpa -n daemon-lab

REGRA:
- Ninguém usa o namespace "default" para este projeto.
- Se precisar criar outro namespace, avisar o grupo.

=====================================================================
5. ENDPOINTS DA API
=====================================================================

A API FastAPI vai expor EXATAMENTE estes endpoints:

MÉTODO  | ROTA                  | O QUE FAZ
--------|-----------------------|--------------------------------
GET     | /processes            | Lista todos os processos
GET     | /processes/{pid}      | Inspeciona um processo pelo PID
POST    | /processes/start      | Inicia um novo processo
DELETE  | /processes/{pid}      | Encerra um processo pelo PID

PARÂMETROS:
  {pid}: número inteiro (int)
  command (no POST): string com o comando a executar
  force (no DELETE): booleano (opcional, padrão false)

EXEMPLOS DE USO:
  curl http://127.0.0.1:8000/processes
  curl http://127.0.0.1:8000/processes/1
  curl -X POST "http://127.0.0.1:8000/processes/start?command=sleep 60"
  curl -X DELETE "http://127.0.0.1:8000/processes/123"

DOCUMENTAÇÃO AUTOMÁTICA:
  http://127.0.0.1:8000/docs

REGRA:
- Ninguém muda o nome das rotas sem avisar o grupo.
- Se mudar, atualizar: API, artigo, slides e documentação.

=====================================================================
6. ESTRUTURA DE PASTAS DO REPOSITÓRIO
=====================================================================

REPOSITÓRIO GITHUB:
  projeto-a3/

ESTRUTURA:
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
  │   ├── interfaces.md       (este documento)
  │   ├── artigo.md
  │   └── slides.pdf
  ├── scripts/                (todos)
  │   ├── build.sh
  │   └── deploy.sh
  └── README.md               (Pessoa 4)

REGRA:
- Cada pessoa só mexe na SUA pasta.
- Se precisar mexer na pasta de outro, avisar antes.
- O README.md é responsabilidade da Pessoa 4, mas todos podem
  sugerir mudanças.

=====================================================================
7. BRANCHES DO GITHUB
=====================================================================

BRANCH PRINCIPAL:
  main (sempre estável, só recebe Pull Requests aprovados)

BRANCHES DE TRABALHO:
  feature/api          (Pessoa 1)
  feature/containers   (Pessoa 2)
  feature/k8s          (Pessoa 3)
  docs/artigo          (Pessoa 4)

FLUXO:
  1. Cada um trabalha na sua branch.
  2. Ao terminar, abre Pull Request para a main.
  3. Outro integrante revisa e faz merge.
  4. Uma vez por semana, todos fazem git pull da main.
  5. Nunca commitar direto na main.

COMANDOS BÁSICOS:
  git checkout -b feature/api     (criar branch)
  git add .                        (adicionar arquivos)
  git commit -m "mensagem"         (commitar)
  git push origin feature/api      (enviar para GitHub)
  git pull origin main             (atualizar da main)

REGRA:
- Ninguém commita direto na main.
- Ninguém mexe na branch do outro.
- Pull Request só é aprovado se os testes passarem.

=====================================================================
8. COMANDOS PADRONIZADOS
=====================================================================

BUILD DAS IMAGENS:
  cd api && docker build -t process-api:latest .
  cd nginx && docker build -t nginx-cron:latest .
  cd cron && docker build -t cron-daemon:latest .

CARREGAR IMAGENS NO MINIKUBE:
  minikube image load process-api:latest
  minikube image load nginx-cron:latest
  minikube image load cron-daemon:latest

APLICAR YAMLS:
  kubectl apply -f k8s/namespace.yaml
  kubectl apply -f k8s/api-deployment.yaml
  kubectl apply -f k8s/api-service.yaml
  kubectl apply -f k8s/nginx-deployment.yaml
  kubectl apply -f k8s/cron-deployment.yaml
  kubectl apply -f k8s/hpa.yaml

VERIFICAR:
  kubectl get pods -n daemon-lab
  kubectl get svc -n daemon-lab
  kubectl get hpa -n daemon-lab

DELETAR TUDO E REFAZER:
  kubectl delete namespace daemon-lab
  minikube delete
  minikube start --driver=docker

REGRA:
- Todos usam os mesmos comandos.
- Se um comando mudar, avisar o grupo.

=====================================================================
9. RESPONSABILIDADES POR PESSOA
=====================================================================

PESSOA 1 — API FastAPI + psutil
  Entregáveis:
    - api/main.py
    - api/requirements.txt
    - api/Dockerfile
    - Prints da API funcionando
  Branch: feature/api
  Prazo: até o final da Semana 2

PESSOA 2 — Containers Nginx e Cron
  Entregáveis:
    - nginx/Dockerfile
    - nginx/entrypoint.sh
    - cron/Dockerfile
    - Prints dos containers rodando
  Branch: feature/containers
  Prazo: até o final da Semana 2

PESSOA 3 — Kubernetes (YAMLs + deploy + self-healing + scaling)
  Entregáveis:
    - k8s/*.yaml (todos os arquivos)
    - Prints do kubectl get pods, svc, hpa
    - Vídeo do self-healing e scaling
  Branch: feature/k8s
  Prazo: até o final da Semana 3

PESSOA 4 — Documentação + Integração + Demonstração
  Entregáveis:
    - docs/artigo.md
    - docs/slides.pdf
    - README.md
    - Roteiro de apresentação
  Branch: docs/artigo
  Prazo: até o final da Semana 4

=====================================================================
10. CRONOGRAMA DE INTEGRAÇÃO
=====================================================================

SEMANA 1:
  - Todos estudam suas partes.
  - Pessoa 4 cria o repositório no GitHub.
  - Pessoa 4 cria este documento de interfaces.

SEMANA 2:
  - Pessoa 1: termina a API e builda a imagem.
  - Pessoa 2: termina os containers e builda as imagens.
  - Pessoa 3: escreve os YAMLs (com imagens genéricas se preciso).
  - Pessoa 4: escreve Introdução e Teoria do artigo.

SEMANA 3 (INTEGRAÇÃO):
  - Reunião do grupo (presencial ou online).
  - Todos juntos: fazer deploy no Minikube.
  - Pessoa 3: demonstrar self-healing e scaling.
  - Pessoa 4: gravar prints e vídeos.

SEMANA 4 (FINALIZAÇÃO):
  - Pessoa 4: finalizar artigo e slides.
  - Todos: ensaiar apresentação (3x cronometrado).
  - Todos: treinar respostas para o professor.
  - Enviar documentação para o professor (25/11/2026).

=====================================================================
11. REGRAS
=====================================================================

1. CADA UM NA SUA BRANCH.
   Ninguém mexe na branch do outro.

2. PULL REQUEST SEMPRE.
   Nada vai direto para a main.

3. INTEGRAÇÃO SEMANAL.
   Toda semana, uma reunião rápida para alinhar.

4. DOCUMENTAR TUDO.
   Prints, vídeos, anotações. Vira o artigo no final.

5. AVISAR SE MUDAR ALGO.
   Se precisar mudar uma interface, avisar TODO o grupo antes.

