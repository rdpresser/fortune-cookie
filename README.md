# Fortune Cookie

Este projeto é uma aplicação web simples que exibe uma mensagem de fortuna ao usuário ao clicar em um botão. A aplicação é composta por um **backend em Rust** que acessa a API externa [https://api.adviceslip.com/advice](https://api.adviceslip.com/advice) para obter as mensagens de fortuna, e um **frontend em Nginx** que serve uma página estática.

Você pode executar a aplicação de duas formas:
- **Docker Compose** (desenvolvimento local rápido)
- **Kubernetes + Argo CD** (GitOps, ambientes dev/prod)

## 📑 Índice

- [📁 Estrutura do Projeto](#-estrutura-do-projeto)
- [🗂️ Hierarquia de Pastas e Arquivos](#️-hierarquia-de-pastas-e-arquivos)
- [🔧 Pré-requisitos](#-pré-requisitos)
- [🐳 Executando com Docker Compose](#-executando-com-docker-compose)
- [☸️ Executando com Kubernetes](#️-executando-com-kubernetes)
- [📜 Verificando os Logs](#-verificando-os-logs)
- [📄 Licença](#-licença)

---

## 📁 Estrutura do Projeto

```
fortune-cookie/
├── backend/                    # Serviço de API em Rust
│   ├── Cargo.toml             # Dependências e metadados do projeto Rust
│   ├── Dockerfile             # Imagem do backend (build multi-stage)
│   └── src/
│       └── main.rs            # Servidor HTTP (rota /fortune)
│
├── frontend/                   # Servidor web estático (Nginx)
│   ├── Dockerfile             # Imagem do frontend (Nginx Alpine)
│   ├── index.html             # Página HTML que consome a API
│   └── nginx.conf             # Configuração do Nginx (proxy para backend)
│
├── k8s-manifests/             # Infraestrutura Kubernetes + GitOps
│   ├── application.yaml       # Argo CD Application (aponta para overlay dev)
│   ├── base/                  # Manifests base (reutilizados entre ambientes)
│   │   ├── backend/
│   │   │   ├── deployment.yaml    # Deployment do backend (probes, recursos)
│   │   │   └── service.yaml       # Service ClusterIP porta 8080
│   │   ├── frontend/
│   │   │   ├── deployment.yaml    # Deployment do frontend (Nginx)
│   │   │   └── service.yaml       # Service ClusterIP porta 80
│   │   ├── ingress/
│   │   │   └── ingress.yaml       # Ingress (paths /, /fortune)
│   │   ├── hpa/
│   │   │   ├── backend-hpa.yaml   # Autoscaling backend (CPU 70%)
│   │   │   └── frontend-hpa.yaml  # Autoscaling frontend (CPU 70%)
│   │   └── kustomization.yaml     # Lista recursos base
│   │
│   └── overlays/              # Customizações por ambiente
│       ├── dev/
│       │   └── kustomization.yaml # Overlay dev (tags, labels environment)
│       └── prod/
│           ├── backend-replicas-patch.yaml   # Aumenta réplicas backend (3)
│           ├── frontend-replicas-patch.yaml  # Aumenta réplicas frontend (3)
│           └── kustomization.yaml            # Overlay prod (patches)
│
├── fortune_logs/              # Logs locais (montados via volumes Docker)
│   ├── backend/
│   └── frontend/
│
├── docker-compose.yaml        # Orquestração Docker local (backend + frontend)
├── README.md                  # Este arquivo
└── LICENSE                    # Licença Apache 2.0
```

---

## 🗂️ Hierarquia de Pastas e Arquivos

### **backend/**
Contém o código do serviço de API em Rust.

| Arquivo | Função |
|---------|--------|
| `Cargo.toml` | Define dependências (actix-web, reqwest, serde, flexi_logger) e metadados do projeto Rust. |
| `Dockerfile` | Build multi-stage: compila o binário Rust e cria imagem final leve (~100MB). |
| `src/main.rs` | Servidor HTTP com rota `GET /fortune` que busca conselhos na API externa e loga operações. |

**Fluxo**: Recebe requisições HTTP → Busca em `api.adviceslip.com` → Retorna JSON `{"message": "..."}`.

---

### **frontend/**
Servidor web estático baseado em Nginx.

| Arquivo | Função |
|---------|--------|
| `Dockerfile` | Cria imagem com Nginx Alpine, copia `index.html` e `nginx.conf` customizado. |
| `index.html` | Página estática com botão "Get Your Fortune". Faz requisições AJAX para `/fortune`. |
| `nginx.conf` | Configuração Nginx: serve HTML em `/` e faz proxy de `/fortune` para o backend. |

**Fluxo**: Usuário acessa `http://localhost:8000` → Clica botão → JavaScript chama `/fortune` → Nginx encaminha para backend → Resposta exibida na página.

---

### **k8s-manifests/**
Infraestrutura como código para Kubernetes (GitOps com Argo CD).

#### **application.yaml**
- **O que é**: Recurso do Argo CD que define qual repositório Git, branch e path monitorar.
- **Função**: Aponta para `k8s-manifests/overlays/dev` (ou prod) e sincroniza automaticamente mudanças do Git no cluster.
- **Uso**: `kubectl apply -f k8s-manifests/application.yaml` cria a Application no Argo CD.

#### **base/** (Manifests fundação)
Contém definições base reutilizadas entre ambientes.

**base/backend/**
| Arquivo | Função |
|---------|--------|
| `deployment.yaml` | Define Deployment do backend: 1 réplica, imagem `localhost:5000/fortune-backend:1.0`, probes HTTP em `/fortune`, recursos CPU/Memory, volume para logs. |
| `service.yaml` | Expõe backend internamente na porta 8080 (ClusterIP). Nome DNS: `fortune-backend`. |

**base/frontend/**
| Arquivo | Função |
|---------|--------|
| `deployment.yaml` | Define Deployment do frontend: 1 réplica, imagem `localhost:5000/fortune-frontend:1.0`, probes HTTP em `/`, volume para cache Nginx. |
| `service.yaml` | Expõe frontend internamente na porta 80 (ClusterIP). Nome DNS: `fortune-frontend`. |

**base/ingress/**
| Arquivo | Função |
|---------|--------|
| `ingress.yaml` | Roteamento externo: `fortune-cookie.local/` → frontend, `fortune-cookie.local/fortune` → backend. |

**base/hpa/**
| Arquivo | Função |
|---------|--------|
| `backend-hpa.yaml` | Autoscaling horizontal: escala backend entre 1-5 réplicas baseado em CPU (target 70%). |
| `frontend-hpa.yaml` | Autoscaling horizontal: escala frontend entre 1-5 réplicas baseado em CPU (target 70%). |

**base/kustomization.yaml**
- **Função**: Lista todos os recursos da pasta `base/` e aplica labels comuns (`app.kubernetes.io/instance: fortune-cookie`).
- **Por quê**: Permite composição (Kustomize) para overlays herdarem essa base.

---

#### **overlays/** (Customizações por ambiente)

**overlays/dev/kustomization.yaml**
- **Função**: Referencia `../../base`, adiciona label `environment: dev`, parametriza tags das imagens.
- **Uso**: `kubectl apply -k k8s-manifests/overlays/dev` aplica versão dev.
- **Benefício**: Pode trocar tag da imagem (ex: `1.0` → `1.1`) sem alterar base.

**overlays/prod/**
| Arquivo | Função |
|---------|--------|
| `backend-replicas-patch.yaml` | Patch estratégico que aumenta réplicas do backend para 3. |
| `frontend-replicas-patch.yaml` | Patch estratégico que aumenta réplicas do frontend para 3. |
| `kustomization.yaml` | Aplica patches acima + label `environment: prod`. |

**Uso**: `kubectl apply -k k8s-manifests/overlays/prod` aplica versão produção (3 réplicas).

---

### **Fluxo GitOps (Kubernetes + Argo CD)**
1. **Commit/Push**: Desenvolvedor altera manifests na branch e faz push.
2. **Argo CD detecta**: Application monitora branch `feature/docker-compose-fix` no GitHub.
3. **Kustomize renderiza**: Combina `base/` + `overlays/dev/` (ou prod).
4. **Sync automático**: Argo aplica mudanças no cluster (prune, selfHeal ativados).
5. **HPAs monitoram**: Escalam pods automaticamente conforme carga (quando metrics-server configurado).

---

### **Por que separar base/overlays?**
- **Base**: Lógica comum (deployments, services, probes, limites).
- **Overlays**: Ajustes por ambiente (tags de imagem, réplicas, labels, futuros patches de TLS/recursos).
- **Vantagem**: Promove mudanças (dev → prod) alterando apenas overlay; evita duplicação de YAMLs.

---

## 🔧 Pré-requisitos

Certifique-se de ter instalado:

### Para Docker Compose
- **Docker Desktop** (Windows/macOS/Linux)
- **Docker Compose** (incluído no Docker Desktop)

### Para Kubernetes
- **k3d** ou **Minikube** (cluster Kubernetes local)
- **kubectl** (CLI do Kubernetes)
- **Argo CD** (opcional, para GitOps)
- **Rust** (opcional, somente se quiser compilar o backend localmente)

---

## 🐳 Executando com Docker Compose

### 📥 Clonando o Repositório
```powershell
git clone https://github.com/rdpresser/fortune-cookie.git
cd fortune-cookie
```

### 🗂 Criando Diretórios de Logs
```powershell
mkdir -p fortune_logs/backend
mkdir -p fortune_logs/frontend
```

### 🛠 Construindo as Imagens

**Backend:**
```powershell
cd backend
docker build -t localhost/fortune-backend:latest .
cd ..
```

**Frontend:**
```powershell
cd frontend
docker build -t localhost/fortune-frontend:latest .
cd ..
```

### 📝 Arquivo docker-compose.yaml

O arquivo `docker-compose.yaml` já está na raiz do projeto:

```yaml
version: "3.9"

services:
  backend:
    image: localhost/fortune-backend:latest
    container_name: fortune-backend
    environment:
      - API_URL=https://api.adviceslip.com/advice
    volumes:
      - ./fortune_logs/backend:/var/log/fortune_backend
    networks:
      - fortune-net

  frontend:
    image: localhost/fortune-frontend:latest
    container_name: fortune-frontend
    ports:
      - "8000:80"
      - "8080:8080"
    volumes:
      - ./fortune_logs/frontend:/var/log/nginx
    networks:
      - fortune-net
    depends_on:
      - backend

networks:
  fortune-net:
    driver: bridge
```

### 🚀 Subindo a Aplicação

**Construir e subir tudo:**
```powershell
docker compose up -d --build
```

**Apenas subir (se imagens já existirem):**
```powershell
docker compose up -d
```

**Ver status:**
```powershell
docker compose ps
```

**Derrubar tudo:**
```powershell
docker compose down
```

### 🌐 Acessando a Aplicação

Abra no navegador:
```
http://localhost:8000
```

Clique no botão **"Get Your Fortune"** para obter uma mensagem da API.

---

## ☸️ Executando com Kubernetes

### 🔧 Pré-requisitos Kubernetes

1. **Criar cluster k3d** (se ainda não tiver):
```powershell
k3d cluster create dev --agents 2 --registry-create k3d-registry.local:5000
```

2. **Instalar Argo CD** (opcional, para GitOps):
```powershell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 🏗️ Build e Push das Imagens

**1. Construir imagens:**
```powershell
docker build -t fortune-backend:1.0 ./backend
docker build -t fortune-frontend:1.0 ./frontend
```

**2. Tagear para o registry local:**
```powershell
docker tag fortune-backend:1.0 localhost:5000/fortune-backend:1.0
docker tag fortune-frontend:1.0 localhost:5000/fortune-frontend:1.0
```

**3. Importar no cluster k3d:**
```powershell
k3d image import localhost:5000/fortune-backend:1.0 localhost:5000/fortune-frontend:1.0 -c dev
```

**Ou fazer push para registry local:**
```powershell
docker push localhost:5000/fortune-backend:1.0
docker push localhost:5000/fortune-frontend:1.0
```

### 🚀 Deploy com Kustomize

**Aplicar overlay de desenvolvimento:**
```powershell
kubectl apply -k k8s-manifests/overlays/dev
```

**Aplicar overlay de produção:**
```powershell
kubectl apply -k k8s-manifests/overlays/prod
```

**Verificar recursos:**
```powershell
kubectl get pods,svc,ingress,hpa -n default
```

### 🎯 Deploy com Argo CD (GitOps)

**1. Aplicar Application:**
```powershell
kubectl apply -f k8s-manifests/application.yaml
```

**2. Verificar status:**
```powershell
kubectl get application -n argocd
kubectl describe application fortune-app -n argocd
```

**3. Acessar UI do Argo CD:**
```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Acesse `https://localhost:8080` (usuário: `admin`, senha obtida via `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`).

### 🌐 Acessando a Aplicação no Kubernetes

**Opção 1: Port-forward**
```powershell
# Frontend
kubectl port-forward svc/fortune-frontend 8000:80 -n default

# Backend direto
kubectl port-forward svc/fortune-backend 8081:8080 -n default
```

Acesse:
- Frontend: `http://localhost:8000`
- Backend: `http://localhost:8081/fortune`

**Opção 2: Ingress (configurar /etc/hosts)**

Adicione ao arquivo `C:\Windows\System32\drivers\etc\hosts`:
```
127.0.0.1 fortune-cookie.local
```

Acesse: `http://fortune-cookie.local/`

### 🔄 Atualizando Imagens

**1. Rebuild:**
```powershell
docker build -t fortune-backend:1.1 ./backend
docker tag fortune-backend:1.1 localhost:5000/fortune-backend:1.1
```

**2. Import:**
```powershell
k3d image import localhost:5000/fortune-backend:1.1 -c dev
```

**3. Atualizar overlay (editar `k8s-manifests/overlays/dev/kustomization.yaml`):**
```yaml
images:
  - name: localhost:5000/fortune-backend
    newTag: "1.1"  # ← alterar aqui
```

**4. Aplicar:**
```powershell
kubectl apply -k k8s-manifests/overlays/dev
```

**5. Restart (forçar pull):**
```powershell
kubectl rollout restart deployment fortune-backend -n default
```

---

## 📜 Verificando os Logs

### Docker Compose

**Logs do Backend:**
```powershell
Get-Content fortune_logs/backend/fortune_backend.log
```

**Logs do Frontend:**
```powershell
Get-Content fortune_logs/frontend/access.log
Get-Content fortune_logs/frontend/error.log
```

**Logs em tempo real (Docker Compose):**
```powershell
docker compose logs -f backend
docker compose logs -f frontend
```

### Kubernetes

**Logs dos Pods:**
```powershell
kubectl logs -f deployment/fortune-backend -n default
kubectl logs -f deployment/fortune-frontend -n default
```

**Ver eventos:**
```powershell
kubectl get events -n default --sort-by='.lastTimestamp'
```

**Descrever pod com problemas:**
```powershell
kubectl describe pod <nome-do-pod> -n default
```

---

## 🔍 Troubleshooting

### ImagePullBackOff no Kubernetes

**Problema**: Pods não conseguem baixar imagem.

**Solução**:
```powershell
# Importar imagens diretamente no cluster
k3d image import localhost:5000/fortune-backend:1.0 localhost:5000/fortune-frontend:1.0 -c dev

# Verificar imagens importadas
docker exec k3d-dev-agent-0 crictl images | Select-String fortune
```

### Frontend com erro "host not found in upstream"

**Problema**: Nginx não consegue resolver o hostname do backend.

**Causa**: Service do backend tem nome diferente do esperado no `nginx.conf`.

**Solução**: Garantir que `nginx.conf` usa `fortune-backend` (nome do Service).

### HPAs mostram `<unknown>` para métricas

**Problema**: Metrics Server não está instalado.

**Solução**:
```powershell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## 📚 Comandos Úteis

### Docker Compose

```powershell
# Build e subir
docker compose up -d --build

# Ver logs
docker compose logs -f

# Status
docker compose ps

# Parar
docker compose stop

# Remover tudo
docker compose down -v

# Reconstruir apenas um serviço
docker compose build backend
docker compose up -d backend
```

### Kubernetes

```powershell
# Aplicar manifests
kubectl apply -k k8s-manifests/overlays/dev

# Ver todos os recursos
kubectl get all -n default

# Deletar tudo do overlay
kubectl delete -k k8s-manifests/overlays/dev

# Port-forward
kubectl port-forward svc/fortune-backend 8081:8080 -n default

# Escalar manualmente
kubectl scale deployment fortune-backend --replicas=3 -n default

# Ver logs de múltiplos pods
kubectl logs -l app.kubernetes.io/component=backend -n default --tail=50

# Executar comando dentro do pod
kubectl exec -it deployment/fortune-backend -n default -- /bin/sh

# Ver manifests renderizados do Kustomize
kubectl kustomize k8s-manifests/overlays/dev
```

---

## 🎯 Workflows Recomendados

### Desenvolvimento Local (Docker Compose)

1. Faça alterações no código (`backend/src/main.rs` ou `frontend/index.html`)
2. Rebuild: `docker compose up -d --build`
3. Teste: `http://localhost:8000`
4. Verifique logs: `docker compose logs -f`

### Desenvolvimento Local (Kubernetes)

1. Altere código
2. Build: `docker build -t fortune-backend:dev ./backend`
3. Import: `k3d image import fortune-backend:dev -c dev`
4. Update deployment (trocar tag para `dev` no kustomization)
5. Apply: `kubectl apply -k k8s-manifests/overlays/dev`
6. Restart: `kubectl rollout restart deployment fortune-backend -n default`

### GitOps (Produção)

1. Desenvolva na branch `feature/*`
2. Teste localmente (compose ou k3d)
3. Merge para `main`
4. CI/CD builda imagens com tag semântica (ex: `1.2.0`)
5. Atualiza `overlays/prod/kustomization.yaml` com nova tag
6. Push → Argo CD sincroniza automaticamente
7. Monitore: UI do Argo CD ou `kubectl get application -n argocd`

---

## 📄 Licença

Este projeto é licenciado sob os termos da licença **Apache 2.0**. Veja o arquivo [LICENSE](LICENSE) para mais detalhes.

---

## 🤝 Contribuindo

Contribuições são bem-vindas! Sinta-se à vontade para:

1. Fazer fork do projeto
2. Criar uma branch para sua feature (`git checkout -b feature/nova-feature`)
3. Commit suas mudanças (`git commit -m 'Adiciona nova feature'`)
4. Push para a branch (`git push origin feature/nova-feature`)
5. Abrir um Pull Request

---

## 📞 Suporte

Se você encontrar algum problema ou tiver dúvidas:

- Abra uma [issue](https://github.com/rdpresser/fortune-cookie/issues) no repositório
- Consulte a documentação do [Argo CD](https://argo-cd.readthedocs.io/)
- Verifique a documentação do [Kustomize](https://kustomize.io/)

---

**Nota**: Este projeto usa Docker e Kubernetes. Os comandos fornecidos são para ambientes Windows (PowerShell) mas podem ser adaptados para Linux/macOS substituindo `Get-Content` por `cat` e ajustando paths.
