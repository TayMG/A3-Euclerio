# Guia da Pessoa 4 — Documentação + Integração + Demonstração

---

## 1. Objetivo

Criar e organizar toda a documentação do projeto:
- Repositório no GitHub.
- Documentos de interfaces e tarefas.
- Artigo final (Introdução, Projeto, Resultados, Conclusão, Anexos).
- Slides da apresentação.
- README do repositório.
- Roteiro da apresentação.
- Coordenar a integração final e ensaiar com o grupo.

---

## 2. O que precisa estar instalado

- WSL2 com Ubuntu
- Git
- Editor de texto (VS Code, nano ou similar)
- Conta no GitHub
- LibreOffice ou PowerPoint (para os slides)
- Navegador web

Se algum desses não estiver instalado, peça ajuda no grupo antes de continuar.

---

## 3. Dependências entre pessoas

Esta guia tem **várias dependências**, mas a maior parte pode ser feita **antes** delas ficarem prontas.

| Passo | Depende de | O que precisa |
|-------|-----------|---------------|
| 5.4 (escrever Introdução e Teoria) | Ninguém | Pode fazer desde o Dia 1 |
| 5.5 (escrever Projeto) | Pessoa 1, 2 e 3 | Prints e descrição das partes |
| 5.6 (escrever Resultados) | Pessoa 3 | Tabela de resultados + prints + vídeos |
| 5.7 (escrever Conclusão) | Pessoa 3 | Resultados para basear a conclusão |
| 5.9 (montar slides) | Todos | Prints e informações de todas as partes |
| 5.11 (integrar tudo) | Todos | Tudo pronto |
| 5.12 (ensaiar) | Todos | Slides prontos |

**Estratégia:** comece pelo que **não depende de ninguém** (passos 5.1 a 5.4). Depois, quando as outras pessoas entregarem, você preenche o resto.

**Aviso:** você é a pessoa que **cobra** as entregas dos outros. Se alguém atrasar, avise o grupo e, se necessário, o professor.

---

## 4. Teoria essencial

### 4.1. O que é um artigo acadêmico

Um artigo é um documento escrito que apresenta um projeto de forma estruturada. Geralmente tem:

| Seção | O que contém |
|-------|--------------|
| Introdução | Contexto, problema, objetivo do trabalho |
| Embasamento teórico | Conceitos usados (daemon, container, Kubernetes, etc.) |
| Projeto | Descrição do que foi construído, arquitetura, prints |
| Resultados | O que foi obtido, dados, tabelas, gráficos |
| Conclusão | Aprendizados, limitações, trabalhos futuros |
| Anexos | Slides, roteiro, arquivos de configuração |

O edital pede **exatamente essa estrutura**. Siga-a.

### 4.2. O que é Markdown

**Markdown** é uma linguagem de marcação simples para formatar texto. É o formato padrão de documentação no GitHub.

**Sintaxe básica:**

```markdown
# Título nível 1
## Título nível 2
### Título nível 3

**negrito**
*itálico*

- item de lista
- outro item

1. item numerado
2. outro item

| Coluna 1 | Coluna 2 |
|----------|----------|
| valor 1  | valor 2  |

`código inline`

```bash
bloco de código
```

[link para algo](https://exemplo.com)

![imagem](caminho/da/imagem.png)
```

### 4.3. O que são slides de apresentação

Slides são a **versão visual** do artigo. Regras:

- **Pouco texto:** no máximo 6 linhas por slide.
- **Muito visual:** diagramas, prints, tabelas.
- **Tempo:** 20 minutos no total (o edital exige).
- **Divisão:** ~15 min de apresentação + ~5 min de perguntas.

**Regra de ouro:** os slides **não são** o artigo. São um apoio visual para quem está falando.

### 4.4. Estrutura recomendada de slides (12 slides)

| Slide | Conteúdo |
|-------|----------|
| 1 | Capa (título, integrantes, matrícula, data) |
| 2 | Introdução (tema, objetivo) |
| 3 | Embasamento teórico (daemon, container, Kubernetes) |
| 4 | Arquitetura do sistema (diagrama) |
| 5 | API FastAPI (endpoints, prints) |
| 6 | Containers (Dockerfiles, prints) |
| 7 | Kubernetes (Deployments, Services, YAMLs) |
| 8 | Self-healing (demonstração) |
| 9 | Scaling (demonstração) |
| 10 | Resultados e métricas |
| 11 | Conclusão e trabalhos futuros |
| 12 | Referências |

### 4.5. Técnicas de apresentação

- **Ensaie 3 vezes** antes do dia.
- **Cronometre** cada ensaio.
- **Não leia os slides:** use-os como apoio.
- **Fale devagar** e olhe para a turma.
- **Prepare respostas** para as perguntas do professor (banco separado).
- **Leve o artigo impresso** para consulta rápida.

---

## 5. Passo a passo

### 5.1. Criar o repositório no GitHub

**Passo 1:** Acesse github.com e faça login.

**Passo 2:** Clique em **New repository**.

**Passo 3:** Preencha:
- **Repository name:** `projeto-a3`
- **Description:** `Projeto A3 - Tema 1: Orquestração de Serviços Daemon com Kubernetes`
- **Visibility:** Private (ou Public, se preferir)
- Marque **Add a README file**.

**Passo 4:** Clique em **Create repository**.

**Passo 5:** Adicione os outros integrantes como colaboradores:
- Vá em **Settings → Collaborators → Add people**.
- Adicione os 3 colegas pelo usuário do GitHub.

**Passo 6:** No seu terminal WSL2, clone o repositório:
```bash
cd ~
git clone https://github.com/<seu-usuario>/projeto-a3.git
cd projeto-a3
```

**Avise o grupo** que o repositório está criado e o link.

### 5.2. Criar a estrutura de pastas

```bash
cd ~/projeto-a3
mkdir -p docs docs/evidencias docs/evidencias/pessoa-1 docs/evidencias/pessoa-2 docs/evidencias/pessoa-3 docs/evidencias/pessoa-4
```

**Estrutura final esperada:**
```
projeto-a3/
├── api/                    (Pessoa 1)
├── nginx/                  (Pessoa 2)
├── cron/                   (Pessoa 2)
├── k8s/                    (Pessoa 3)
├── docs/                   (Pessoa 4)
│   ├── interfaces.md
│   ├── tarefas.md
│   ├── artigo.md
│   ├── slides.pdf
│   ├── roteiro-apresentacao.md
│   └── evidencias/
│       ├── pessoa-1/
│       ├── pessoa-2/
│       ├── pessoa-3/
│       └── pessoa-4/
└── README.md
```

### 5.3. Criar os documentos de interface e tarefas

**Passo 1:** Crie `docs/interfaces.md` com o conteúdo do documento de interfaces (você já tem).

**Passo 2:** Crie `docs/tarefas.md` com o conteúdo da divisão de tarefas (você já tem).

**Passo 3:** Suba para o GitHub:
```bash
git add docs/interfaces.md docs/tarefas.md
git commit -m "Adiciona documentos de interfaces e tarefas"
git push origin main
```

### 5.4. Escrever Introdução e Embasamento Teórico

> ✅ **Este passo não depende de ninguém.** Pode ser feito desde o Dia 1.

Crie `docs/artigo.md` com a estrutura inicial:

```markdown
# Orquestração de Serviços Daemon com Kubernetes e API de Gerenciamento de Processos

**Integrantes:**
- Nome 1 — Matrícula
- Nome 2 — Matrícula
- Nome 3 — Matrícula
- Nome 4 — Matrícula

**Data:** ____/____/2026

---

## 1. Introdução

[Escrever aqui]

---

## 2. Embasamento Teórico

[Escrever aqui]

---

## 3. Projeto

[Escrever aqui]

---

## 4. Resultados

[Escrever aqui]

---

## 5. Conclusão

[Escrever aqui]

---

## 6. Anexos

[Escrever aqui]
```

**O que escrever na Introdução (2–3 parágrafos):**
- Contextualize: containers e Kubernetes estão cada vez mais presentes no mercado.
- Problema: gerenciar processos em ambientes distribuídos é desafiador.
- Objetivo: construir um ambiente com 3 containers orquestrados pelo Kubernetes, com uma API que gerencia processos.
- Justificativa: aplicar conceitos de Organização de Computadores e Sistemas Operacionais.

**O que escrever no Embasamento Teórico (3–5 páginas):**
- **Daemon:** definição, características, exemplos.
- **Container vs. VM:** diferenças, vantagens.
- **Docker:** imagem, container, Dockerfile.
- **Kubernetes:** arquitetura, componentes, Pod, Deployment, Service, ReplicaSet.
- **Self-healing:** como funciona.
- **Scaling e HPA:** como funciona.
- **FastAPI:** o que é, para que serve.
- **psutil:** o que é, para que serve.
- **PID, SIGTERM, SIGKILL:** conceitos do Linux.

Use o material teórico das guias 1, 2 e 3 como base. Copie, cole e adapte.

### 5.5. Escrever a seção Projeto

> ⚠️ **Depende da Pessoa 1, 2 e 3.** Você precisa dos prints e descrições.

**O que escrever (3–4 páginas):**

- **Descrição geral:** o que o projeto faz.
- **Arquitetura:** diagrama mostrando os 3 containers, o Service, o namespace.
- **API FastAPI:**
  - Endpoints (tabela).
  - Prints do `/docs` e das respostas.
  - Explicação do uso de psutil.
- **Containers:**
  - Dockerfile do Nginx + Cron.
  - Dockerfile do Cron separado.
  - Prints do `docker ps`.
- **Kubernetes:**
  - YAMLs principais (Deployment, Service, HPA).
  - Prints do `kubectl get pods/svc/hpa`.
  - Explicação do `hostPID`.

**Peça à Pessoa 1, 2 e 3:**
- Prints das suas partes (eles salvam em `docs/evidencias/pessoa-X/`).
- Descrição curta do que fizeram.

### 5.6. Escrever a seção Resultados

> ⚠️ **Depende da Pessoa 3.** Você precisa da tabela de resultados e dos vídeos.

**O que escrever (2–3 páginas):**

- **Self-healing:**
  - Como foi demonstrado (deletar pod → recriação automática).
  - Tempo de recriação.
  - Print ou frame do vídeo.
- **Scaling:**
  - Como foi demonstrado (gerar carga → réplicas aumentam).
  - Tabela com réplicas antes/depois.
  - Print do `kubectl get hpa`.
- **Análise:**
  - O Kubernetes se comportou como esperado?
  - Houve algum problema?
  - O que poderia ser melhorado?

Use a tabela de resultados que a Pessoa 3 montou.

### 5.7. Escrever a Conclusão

> ⚠️ **Depende da Pessoa 3.** Precisa dos resultados.

**O que escrever (1–2 páginas):**

- **Aprendizados:** o que o grupo aprendeu sobre Kubernetes, containers, FastAPI, etc.
- **Limitações:** o que não foi possível fazer.
- **Trabalhos futuros:**
  - Adicionar mais serviços.
  - Usar Helm para gerenciar os YAMLs.
  - Integrar CI/CD.
  - Usar um cluster real (não só Minikube).
  - Implementar monitoramento com Prometheus + Grafana.

### 5.8. Escrever os Anexos

**O que incluir:**

- Slides (link ou cópia).
- Roteiro de instalação e configuração.
- Prompt e nome usados na LLM (se aplicável).
- Arquivos de configuração (YAMLs, Dockerfiles).

**Roteiro de instalação:**
- Liste os passos para reproduzir o projeto do zero.
- Inclua os comandos exatos.
- Referencie as guias de cada pessoa.

### 5.9. Montar os slides

> ⚠️ **Depende de todos.** Precisa dos prints e informações de todas as partes.

**Ferramenta:** PowerPoint, LibreOffice Impress ou Google Slides.

**Estrutura:** 12 slides (ver seção 4.4).

**Regras de design:**
- Fonte grande (mínimo 24pt).
- Pouco texto por slide.
- Use os prints das evidências.
- Use um diagrama para a arquitetura.
- Cores consistentes.

**Salve como PDF:** `docs/slides.pdf`.

### 5.10. Escrever o README.md

Crie `README.md` na raiz com:

```markdown
# Projeto A3 — Tema 1

**Orquestração de Serviços Daemon com Kubernetes e API de Gerenciamento de Processos**

## Integrantes

| Nome | Matrícula |
|------|-----------|
| ... | ... |

## Descrição

Ambiente com 3 containers (API FastAPI, Nginx com cron, Cron separado) orquestrados pelo Kubernetes. Demonstra self-healing e scaling.

## Estrutura do repositório

(cole a estrutura de pastas)

## Como rodar

1. Buildar as imagens:
   ```bash
   cd api && docker build -t process-api:latest .
   cd ../nginx && docker build -t nginx-cron:latest .
   cd ../cron && docker build -t cron-daemon:latest .
   ```

2. Iniciar o Minikube:
   ```bash
   minikube start --driver=docker
   minikube image load process-api:latest
   minikube image load nginx-cron:latest
   minikube image load cron-daemon:latest
   ```

3. Aplicar os YAMLs:
   ```bash
   kubectl apply -f k8s/
   ```

4. Acessar a API:
   ```bash
   curl http://$(minikube ip):30080/processes
   ```

## Documentação

- [Interfaces](docs/interfaces.md)
- [Tarefas](docs/tarefas.md)
- [Artigo](docs/artigo.md)
- [Slides](docs/slides.pdf)
```

### 5.11. Coordenar a integração

> ⚠️ **Depende de todos.** Marque uma reunião na Semana 3.

**Passos:**
1. Combine data e hora com o grupo (presencial ou online).
2. Peça que cada um traga sua parte pronta.
3. Faça o deploy completo no Minikube.
4. Demonstre self-healing e scaling.
5. Grave prints e vídeos finais.
6. Preencha a tabela de resultados.

Se algo der errado, ajude a resolver ou acione quem é responsável.

### 5.12. Escrever o roteiro de apresentação

Crie `docs/roteiro-apresentacao.md` com:

```markdown
# Roteiro da Apresentação

**Duração total:** 20 minutos

## Divisão do tempo

| Tempo | Quem fala | Conteúdo |
|-------|-----------|----------|
| 0–2 min | Pessoa 4 | Introdução e objetivo |
| 2–5 min | Pessoa 4 | Embasamento teórico |
| 5–8 min | Pessoa 1 | API FastAPI |
| 8–10 min | Pessoa 2 | Containers Nginx e Cron |
| 10–14 min | Pessoa 3 | Kubernetes + self-healing + scaling |
| 14–17 min | Pessoa 4 | Resultados e conclusão |
| 17–20 min | Todos | Perguntas do professor |

## Perguntas prováveis do professor

(consultar o banco de perguntas em `docs/banco-perguntas.md`)

## Checklist do dia

- [ ] Notebook carregado
- [ ] Minikube rodando
- [ ] Slides em PDF no pen drive
- [ ] Artigo impresso
- [ ] Água
```

### 5.13. Ensaiar a apresentação

**Faça 3 ensaios completos:**
1. Ensaio 1: apenas para ver se o tempo bate.
2. Ensaio 2: ajustar falas e ritmo.
3. Ensaio 3: simular o dia (com perguntas).

**Cronometre cada ensaio.** Se passar de 20 min, corte conteúdo.

**Treine as perguntas** do banco de perguntas em voz alta.

### 5.14. Enviar para o professor

**Prazo:** 25/11/2026.

**E-mail:** euclerio.filho@ulife.com.br

**Anexos:**
- Artigo em PDF ou DOCX.
- Slides em PDF.
- Roteiro de instalação e configuração.
- Prompt e nome usados na LLM (se aplicável).
- Arquivos de configuração (YAMLs, Dockerfiles) — ou link do GitHub.

**Assunto:** `Projeto A3 — Tema 1 — [Nome do grupo]`

**Corpo:** texto curto explicando que é a entrega e listando os anexos.

---

## 6. Modelos prontos

### 6.1. Estrutura do artigo

```markdown
# Título do Projeto

**Integrantes:** ...

## 1. Introdução
- Contexto
- Problema
- Objetivo
- Justificativa

## 2. Embasamento Teórico
- Daemon
- Container vs. VM
- Docker
- Kubernetes
- Self-healing
- Scaling e HPA
- FastAPI
- psutil

## 3. Projeto
- Descrição geral
- Arquitetura
- API FastAPI
- Containers
- Kubernetes

## 4. Resultados
- Self-healing
- Scaling
- Análise

## 5. Conclusão
- Aprendizados
- Limitações
- Trabalhos futuros

## 6. Anexos
- Slides
- Roteiro de instalação
- Arquivos de configuração
```

### 6.2. Estrutura dos slides

(Já descrita na seção 4.4)

### 6.3. Roteiro de instalação

```markdown
# Roteiro de Instalação e Configuração

## 1. Ambiente
- WSL2 com Ubuntu
- Docker Desktop
- Minikube
- kubectl
- Python 3.10+
- Git

## 2. Clonar o repositório
git clone https://github.com/<usuario>/projeto-a3.git
cd projeto-a3

## 3. Buildar as imagens
cd api && docker build -t process-api:latest .
cd ../nginx && docker build -t nginx-cron:latest .
cd ../cron && docker build -t cron-daemon:latest .

## 4. Iniciar o Minikube
minikube start --driver=docker
minikube image load process-api:latest
minikube image load nginx-cron:latest
minikube image load cron-daemon:latest

## 5. Aplicar os YAMLs
kubectl apply -f k8s/

## 6. Acessar a API
curl http://$(minikube ip):30080/processes
```

---

## 7. Erros comuns e soluções

| Erro | Causa | Solução |
|------|-------|---------|
| Não consegue adicionar colaborador | Repositório privado sem plano | Tornar público temporariamente |
| Conflito no git | Dois editando o mesmo arquivo | Combinar quem edita o quê |
| Slides não abrem | Formato incompatível | Salvar como PDF |
| Artigo muito curto | Falta conteúdo | Mínimo 5 páginas com prints e tabelas |
| Pessoa não entrega a parte | Falta de compromisso | Avisar no grupo e, se necessário, o professor |
| Tempo de apresentação estourou | Conteúdo demais | Cortar slides ou acelerar falas |
| Professor faz pergunta que ninguém sabe | Falta de preparo | Consultar o banco de perguntas |

---

## 8. Próximos passos

Depois de terminar:

1. Confirme que todos enviaram suas partes.
2. Confirme que todos os prints e vídeos estão em `docs/evidencias/`.
3. Confirme que o artigo está completo.
4. Confirme que os slides estão em PDF.
5. Confirme que o roteiro está pronto.
6. Envie tudo para o professor até 25/11/2026.
