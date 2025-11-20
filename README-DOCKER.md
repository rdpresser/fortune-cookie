# Fortune Cookie – Docker Compose Version

Este projeto é uma aplicação simples composta por:
- **Backend em Rust**, que obtém mensagens de fortuna da API https://api.adviceslip.com/advice.
- **Frontend em Nginx**, que exibe a página web para o usuário.

A versão abaixo substitui totalmente os comandos Podman pelo uso de **Docker** e contém um **docker-compose.yml** completo para facilitar build, execução, logs e volumes.

---

## 📁 Estrutura do Projeto
```
.
├── backend
│   ├── Cargo.lock
│   ├── Cargo.toml
│   ├── Dockerfile
│   └── src
│       └── main.rs
├── fortune_logs
│   ├── backend
│   │   └── fortune_backend.log
│   └── frontend
│       ├── access.log
│       └── error.log
└── frontend
    ├── Dockerfile
    ├── index.html
    └── nginx.conf
```

---

## 🔧 Pré-requisitos
Certifique-se de ter instalado:
- **Docker Desktop** (Windows/macOS/Linux)
- **Rust** (opcional, somente se quiser compilar o backend localmente)

---

## 📥 Clonando o Repositório
```powershell
git clone https://github.com/seu-usuario/fortune-cookie.git
cd fortune-cookie
```
> Substitua `seu-usuario` pelo seu nome real no GitHub, se aplicável.

---

## 🗂 Criando Diretórios de Logs
```powershell
mkdir -p fortune_logs/backend
mkdir -p fortune_logs/frontend
```

---

## 🛠 Construindo as Imagens Docker

### 🔹 Construir o Backend
```powershell
cd backend
docker build -t localhost/fortune-backend:latest .
```

### 🔹 Construir o Frontend
```powershell
cd ../frontend
docker build -t localhost/fortune-frontend:latest .
```

Volte para a raiz do projeto:
```powershell
cd ..
```

---

# 🐳 Docker Compose
Abaixo está o arquivo **docker-compose.yml** recomendando para executar todo o ambiente com apenas um comando.

Crie um arquivo chamado **docker-compose.yml** na raiz:

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

networks:
  fortune-net:
    driver: bridge
```

---

# 🚀 Executando a Aplicação com Compose

### 🔹 Subir tudo
```powershell
docker compose up -d
docker compose up -d --build
```

### 🔹 Derrubar tudo
```powershell
docker compose down
```

### 🔹 Ver status dos containers
```powershell
docker compose ps
```

---

# 🌐 Acessando a Aplicação
Abra no navegador:
```
http://localhost:8000
```
Clique no botão **Get Your Fortune** para obter uma mensagem da API.

---

# 📜 Logs

### 🔹 Logs do Backend
```powershell
Get-Content fortune_logs/backend/fortune_backend.log
```

### 🔹 Logs do Frontend
```powershell
Get-Content fortune_logs/frontend/access.log
Get-Content fortune_logs/frontend/error.log
```

---

# 📄 Licença
Este projeto segue a licença **Apache 2.0**.

---

Se quiser adicionar um script PowerShell para build+up automático, posso gerar também!

