# 📐 ANÁLISE DE ARQUITETURA - ObraConnect

## 🎯 Visão Geral do Sistema

ObraConnect é um **Marketplace de Serviços de Construção** que conecta prestadores de serviços com clientes que necessitam de reformas e construções.

### Tipo de Arquitetura: Cliente-Servidor (Web Full Stack)

```
┌─────────────────────┐
│   FRONTEND (Web)    │  <- HTML + CSS + JavaScript Vanilla
│  - Interface        │
│  - Login/Registro   │
│  - Listar Serviços  │
│  - Detalhes         │
└──────────┬──────────┘
           │ HTTP/FETCH
           │
┌──────────▼──────────┐
│   BACKEND (API)     │  <- Node.js + Express
│  - Autenticação     │
│  - Rotas REST       │
│  - Lógica Negócio   │
└──────────┬──────────┘
           │ SQL
┌──────────▼──────────┐
│   DATABASE          │  <- MySQL
│  - Usuários         │
│  - Serviços         │
│  - Avaliações       │
└─────────────────────┘
```

---

## 🏗️ Componentes Principais

### 1. **FRONTEND (client-side)**
- **Local**: `frontend/`
- **Tecnologia**: HTML5 + CSS3 + JavaScript Vanilla
- **Framework CSS**: Bootstrap 5 (via CDN)
- **Padrão**: SPA (Single Page Application) sem framework frontend

**Arquivos chave:**
- `index.html` - Página principal com navbar e listagem de serviços
- `pages/login.html` - Tela de login
- `pages/registro.html` - Tela de registro
- `pages/cadastrar-servico.html` - Criar novo serviço
- `pages/detalhes-servico.html` - Visualizar detalhes e avaliações
- `pages/meu-perfil.html` - Perfil do usuário
- `js/api.js` - Funções para requisições HTTP
- `js/auth.js` - Gerenciamento de autenticação e login/logout
- `js/servicos.js` - Gerenciamento de serviços e UI

### 2. **BACKEND (server-side)**
- **Local**: `backend/`
- **Runtime**: Node.js
- **Framework**: Express.js
- **Puerto**: 3001 (configurável via `.env`)
- **Padrão**: REST API

**Estrutura:**
```
backend/
├── index.js                    # Arquivo principal, configuração do Express
├── package.json               # Dependências e scripts
├── config/
│   ├── database.js           # Pool de conexões MySQL
│   └── upload.js             # Multer para upload de imagens
├── middlewares/
│   └── autenticacao.js       # JWT verification middleware
├── routes/
│   ├── authRoutes.js         # Rotas de autenticação
│   ├── servicoRoutes.js      # Rotas de serviços CRUD
│   └── avaliacaoRoutes.js    # Rotas de avaliações
└── uploads/                   # Pasta para imagens enviadas
```

### 3. **DATABASE (data layer)**
- **Sistema**: MySQL/MariaDB
- **Nome**: `obraconnect_db`
- **Padrão**: Relacional com chaves estrangeiras

---

## 🔄 Fluxo de Requisição

### Exemplo: Listar Serviços

```
1. Frontend (index.html)
   └─> Chama: carregarServicos() [servicos.js]
        ├─> Chama: fazerRequisicao() [api.js]
        │   └─> fetch("http://localhost:3001/api/servicos")
        │
        └─> GET /api/servicos
             │
             └─> Backend [servicoRoutes.js]
                  ├─> Query: SELECT s.*, u.nome_usuario, AVG(avaliacoes)...
                  └─> Database [obraconnect_db]
                       └─> Retorna array de serviços com:
                           - Dados do serviço
                           - Dados do prestador
                           - Nota média calculada
                           - Total de avaliações
             │
             └─> Resposta JSON
                  └─> Frontend renderiza cards com criarCardServico()
```

---

## 🔐 Arquitetura de Segurança

### JWT (JSON Web Token)

1. **Registro**: Usuário cria conta e senha é criptografada com bcryptjs
2. **Login**: Backend retorna JWT válido por tempo limitado
3. **Requisição Protegida**: Cliente envia token no header `Authorization: Bearer TOKEN`
4. **Middleware**: Backend valida JWT antes de processar rota protegida
5. **Expiração**: Token expira após tempo configurado, força novo login

```
Login → Gera JWT → Armazena no localStorage → 
Cada requisição → Valida JWT → Permite acesso
```

---

## 📊 Fluxo de Dados - Autenticação

```
REGISTRO
┌──────────────┐
│ Frontend     │
│ (registro.   │
│  html)       │
└──────┬───────┘
       │ POST /api/auth/registro
       │ {nome, email, senha, login}
       │
       ▼
┌──────────────┐
│ Backend      │
│ (authRoutes) │
│ (verificar  │
│  dados)      │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Criptografar │
│ (bcryptjs)   │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Salvar no    │
│ Database     │
│ (INSERT)     │
└──────┬───────┘
       │
       └─► Retorna: id_usuario


LOGIN
┌──────────────┐
│ Frontend     │
│ (login.html) │
└──────┬───────┘
       │ POST /api/auth/login
       │ {login/email, senha}
       │
       ▼
┌──────────────┐
│ Backend      │
│ (authRoutes) │
│ (localizar   │
│  usuário)    │
└──────┬───────┘
       │ Comparar senha com bcryptjs
       │
       ▼
┌──────────────┐
│ Gerar JWT    │
│ (valido por  │
│  7 dias)     │
└──────┬───────┘
       │
       └─► {token, usuário}
            ↓ salva no localStorage
           Pronto para usar!
```

---

## 🌍 Tecnologias Utilizadas

### Backend
- **Node.js**: Runtime JavaScript
- **Express.js**: Framework web minimalista
- **mysql2/promise**: Driver assíncrono com pool de conexões
- **jsonwebtoken**: Criar e validar JWT
- **bcryptjs**: Hash seguro de senhas
- **multer**: Middleware para upload de arquivos
- **cors**: Middleware para permitir requisições cross-origin
- **dotenv**: Variáveis de ambiente

### Frontend
- **HTML5**: Estrutura semântica
- **CSS3**: Estilização
- **Bootstrap 5**: Framework CSS responsivo
- **Bootstrap Icons**: Ícones SVG
- **JavaScript Vanilla**: Lógica sem frameworks

### Database
- **MySQL**: Sistema de gerenciamento de banco de dados relacional

---

## 📦 Dependências do Projeto

### Backend (package.json)
```json
{
  "bcryptjs": "^2.4.3",      // Criptografia de senhas
  "cors": "^2.8.5",          // CORS middleware
  "dotenv": "^16.4.5",       // Variáveis de ambiente
  "express": "^4.18.2",      // Framework web
  "jsonwebtoken": "^9.0.3",  // JWT
  "multer": "^1.4.5-lts.1",  // Upload de arquivos
  "mysql2": "^3.6.5"         // Driver MySQL
}
```

### Frontend
- Nenhuma dependência NPM (usa CDN para Bootstrap)
- Bootstrap 5.3.0 (via CDN)
- Bootstrap Icons 1.11.0 (via CDN)

### Database
- MySQL 10.4.32-MariaDB (ou compatível)
- phpMyAdmin para gerenciamento

---

## 🎯 Funcionalidades Principais

### 1. Autenticação & Usuários
- ✅ Registro de novo usuário
- ✅ Login com email/login
- ✅ Logout
- ✅ Perfil de usuário
- ✅ Tornar-se prestador de serviços

### 2. Serviços
- ✅ Listar todos os serviços (público)
- ✅ Ver detalhes de serviço (público)
- ✅ Criar novo serviço (prestador)
- ✅ Editar serviço (prestador)
- ✅ Deletar serviço (prestador)
- ✅ Upload de imagem do serviço

### 3. Avaliações
- ✅ Deixar avaliação após contratar
- ✅ Visualizar avaliações de um serviço
- ✅ Calcular nota média automática
- ✅ Ver histórico de minhas avaliações
- ✅ Deletar avaliação própria

### 4. Categorias
- ✅ 30+ categorias pré-cadastradas
- ✅ Associar categoria ao serviço

---

## 💾 Estados da Aplicação

### Estado Global (Frontend)
```javascript
// auth.js
let usuarioLogado = null;        // Dados do usuário autenticado
let tokenAtual = null;           // JWT token

// localStorage
localStorage.token              // JWT token persistido
localStorage.usuario            // Dados do usuário em JSON
```

### Sessão (Backend)
- JWT contém: id, nome, email, tipo_usuario
- Validade: Configurável (padrão 7 dias)
- Armazenado: No cliente (localStorage)

---

## 🔄 Padrões Arquiteturais

### 1. **MVC (Model-View-Controller)**
- **View**: Frontend HTML + CSS + JS
- **Controller**: Routes no Backend
- **Model**: Database struktura

### 2. **REST API**
- Método HTTP + Endpoint = Ação
- GET `/api/servicos` - Listar
- POST `/api/servicos` - Criar
- PUT `/api/servicos/:id` - Editar
- DELETE `/api/servicos/:id` - Deletar

### 3. **Modular**
- Rotas separadas por domínio (auth, services, evaluations)
- Middlewares reutilizáveis
- Funções genéricas no frontend (api.js, auth.js)

---

## 📈 Escalabilidade

### Pontos de Melhoria
1. **Frontend**: Migrar para React/Vue para melhor mantenibilidade
2. **Backend**: Usar TypeScript para type safety
3. **Database**: Índices em estrangeiras (já parcialmente feito)
4. **Cache**: Redis para dados frequentes
5. **Autenticação**: Verificação de email dupla
6. **Arquivo**: Integrar AWS S3 ao invés de armazenar localmente

---

## 🚀 Resumo da Arquitetura

| Aspecto | Tecnologia | Razão |
|---------|-----------|-------|
| **Frontend** | HTML + JS Vanilla | Simplicidade, sem dependências |
| **Backend** | Node.js + Express | JavaScript full-stack |
| **Banco** | MySQL | Dados relacionados, confiável |
| **Autenticação** | JWT + bcryptjs | Sem estado, seguro |
| **Upload** | Multer | Nativo Node.js |
| **CSS** | Bootstrap | Responsivo rápido |

---

Esta é a base arquitetural do ObraConnect! Um sistema moderno, seguro e escalável para marketplace de serviços de construção.
