# 📚 GUIA PASSO A PASSO - 01: CONFIGURAR AMBIENTE DE DESENVOLVIMENTO

## 🎯 Objetivo

Preparar o ambiente para desenvolver o ObraConnect do zero. Você terá:
- Node.js + npm configurado
- MySQL/MariaDB rodando
- Variáveis de ambiente prontas

---

## 📋 Pré-requisitos

### 1. Instalar Node.js

**O que é?** Runtime JavaScript para executar código JS fora do navegador

**Por quê?** Express (backend) roda em Node.js

**Como instalar:**

1. Acesse: https://nodejs.org/
2. Baixe versão **LTS** (recomendada para produção)
3. Execute instalador
4. Após instalar, verifique:

```powershell
# Terminal/CMD
node --version      # Deve retornar v18.x.x ou superior
npm --version       # Deve retornar 9.x.x ou superior
```

**npm?** Node Package Manager - gerenciador de dependências JavaScript

---

### 2. Instalar MySQL/MariaDB

**O que é?** Banco de dados relacional

**Por quê?** Armazenar usuários, serviços, avaliações

**Opção A: Visual - MySQL Community Edition**

1. Acesse: https://dev.mysql.com/downloads/mysql/
2. Baixe Windows MSI Installer
3. Execute com "Developer Default"
4. Configurações:
   - Port: 3306 (padrão)
   - Username: root
   - Password: sua_senha (ou deixe em branco para dev)
5. Finalize instalação

**Opção B: Visual - XAMPP (com phpMyAdmin)**

1. Acesse: https://www.apachefriends.org/
2. Baixe XAMPP
3. Execute
4. Ative MySQL e Apache
5. Abra phpMyAdmin em http://localhost/phpmyadmin

**Opção C: Terminal (se já tem instalado)**

```powershell
# Verificar se MySQL está rodando
mysql --version

# Conectar ao MySQL
mysql -u root -p
# Digitar senha
```

---

### 3. Instalar Gerenciador de Banco (phpMyAdmin ou CLI)

**phpMyAdmin** (Visual):
- Interface web para gerenciar bancos
- Vem com XAMPP
- Acesso: http://localhost/phpmyadmin

**CLI (Terminal)** (Recomendado para dev):
```powershell
# Conectar
mysql -u root -p

# Listar bancos
SHOW DATABASES;

# Criar banco
CREATE DATABASE obraconnect_db;

# Usar banco
USE obraconnect_db;

# Ver tabelas
SHOW TABLES;

# Sair
EXIT;
```

---

## 🔧 Configurar Projeto Backend

### Passo 1: Criar Pastas

```powershell
# Terminal da sua máquina
cd C:\Users\Seu_Usuario\Documents

# Criar pasta do projeto
mkdir projeto-obraConnect
cd projeto-obraConnect

# Criar pasta do backend
mkdir backend
cd backend
```

### Passo 2: Inicializar Node.js

```powershell
# Ainda em backend/
npm init -y
```

**O que faz?** Cria `package.json` com configurações padrão

**Editando package.json:**

```json
{
  "name": "obraconnect-backend",
  "version": "1.0.0",
  "description": "Backend ObraConnect",
  "main": "index.js",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {},
  "devDependencies": {}
}
```

**Scripts explicados:**
- `npm start` → Executa com `node index.js`
- `npm run dev` → Executa com `nodemon` (reinicia ao mudar arquivo)

### Passo 3: Instalar Dependências

```powershell
# Instalar Express
npm install express

# Instalar CORS (permitir requisições frontend)
npm install cors

# Instalar MySQL2
npm install mysql2

# Instalar JWT
npm install jsonwebtoken

# Instalar bcryptjs (criptografia de senha)
npm install bcryptjs

# Instalar Multer (upload de arquivo)
npm install multer

# Instalar dotenv (variáveis de ambiente)
npm install dotenv

# Instalar nodemon em dev
npm install -D nodemon
```

**Resultado:** `package.json` com todas dependências

```json
{
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "mysql2": "^3.6.5",
    "jsonwebtoken": "^9.0.3",
    "bcryptjs": "^2.4.3",
    "multer": "^1.4.5-lts.1",
    "dotenv": "^16.4.5"
  },
  "devDependencies": {
    "nodemon": "^3.0.2"
  }
}
```

---

## 🔐 Criar Arquivo de Variáveis de Ambiente

### Passo 1: Criar `.env`

```powershell
# Ainda em backend/
# Usar Notepad ou Editor preferido

# Criar arquivo .env
type nul > .env
```

### Passo 2: Preencher `.env`

```bash
# Servidor
PORT=3001
NODE_ENV=development

# Banco de dados
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=          # Deixar em branco se não tem senha
DB_NAME=obraconnect_db
DB_PORT=3306

# Autenticação JWT
JWT_SECRET=sua_chave_secreta_muito_segura_123456
JWT_EXPIRY=7d

# Upload
MAX_FILE_SIZE=5242880  # 5MB em bytes
```

**Por quê .env?**

```
❌ Salvar credenciais no código
// config.js
const dbPassword = "senha123";

Problema: Se colocar no GitHub, todos veem a senha!

✅ Salvar em .env (não versionado no Git)
// .env
DB_PASSWORD=senha123

// .gitignore
.env  # Arquivo não entra no Git

// config.js
const dbPassword = process.env.DB_PASSWORD;

Benefício: Código fica limpo, credenciais seguras
```

### Passo 3: Adicionar ao `.gitignore`

```powershell
# Criar .gitignore
type nul > .gitignore
```

Adicionar:
```
node_modules/
.env
.env.local
.env.*.local
*.log
.DS_Store
/uploads
```

---

## 📂 Estrutura de Pastas Inicial

```
backend/
├── package.json           # Dependências
├── package-lock.json      # Locks de versão (auto gerado)
├── .env                   # Variáveis de ambiente
├── .gitignore            # Arquivos ignorados no Git
├── index.js              # Arquivo principal (criar depois)
├── config/               # Configurações
│   ├── database.js       # Conexão MySQL
│   └── upload.js         # Multer config
├── middlewares/          # Middlewares
│   └── autenticacao.js   # JWT middleware
├── routes/               # Rotas da API
│   ├── authRoutes.js     # Auth endpoints
│   ├── servicoRoutes.js  # Service endpoints
│   └── avaliacaoRoutes.js # Review endpoints
├── uploads/              # Pasta de imagens (criar automático)
└── node_modules/         # Dependências instaladas (auto)
```

---

## ✅ Checklist de Conclusão

- [ ] Node.js instalado e verificado
- [ ] MySQL instalado e rodando
- [ ] Pasta `backend/` criada
- [ ] `npm init -y` executado
- [ ] Todas dependências instaladas
- [ ] `.env` criado com variáveis
- [ ] `.gitignore` criado
- [ ] Estrutura de pastas criada

---

## 🧪 Testar Instalação

```powershell
# Em backend/
npm list

# Deve mostrar:
# obraconnect-backend@1.0.0
# ├── express@4.18.2
# ├── cors@2.8.5
# ├── mysql2@3.6.5
# ├── jsonwebtoken@9.0.3
# ├── bcryptjs@2.4.3
# ├── multer@1.4.5-lts.1
# └── dotenv@16.4.5
```

```powershell
# Testar Node.js
node -e "console.log('Node.js funcionando!')"
# Output: Node.js funcionando!
```

```powershell
# Testar MySQL
mysql -u root -p
# Digitar senha (ou Enter se não tem)
# mysql> SELECT 1;
# mysql> EXIT;
```

---

## 🚀 Próximos Passos

Agora seu ambiente está preparado! Próxima etapa:
- Criar arquivos de configuração (database.js, upload.js)
- Criar middleware de autenticação
- Criar arquivo principal (index.js)
- Definir rotas da API

---

## 💡 Dicas

**Problema: MySQL não inicia**
```powershell
# Verificar status (Windows CMD)
mysql.server status

# Iniciar manualmente
mysql.server start

# Ou usar XAMPP/phpMyAdmin
```

**Problema: Port 3001 já em uso**
```powershell
# Mudar em .env
PORT=3002

# Ou liberar port
# Encontrar processo: netstat -ano | findstr :3001
# Matar processo: taskkill /PID numero /F
```

**Problema: Módulos não encontrados**
```powershell
# Reinstalar dependências
npm install

# Verificar se node_modules existe
ls node_modules
```

---

✅ **Seu ambiente está 100% pronto para começar!**
