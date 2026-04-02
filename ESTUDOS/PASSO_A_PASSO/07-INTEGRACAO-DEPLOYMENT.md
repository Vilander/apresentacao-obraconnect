# 📚 GUIA PASSO A PASSO - 07: INTEGRAÇÃO E DEPLOYMENT

## 🎯 Objetivo

Conectar frontend e backend, testar sistema completo e fazer deploy

---

## 🔄 Passo 1: Integração Frontend-Backend

### Estrutura Final de Pastas

```
projeto-obraConnect/
├── backend/                # Servidor API
│   ├── index.js
│   ├── package.json
│   ├── .env
│   ├── config/
│   ├── middlewares/
│   ├── routes/
│   └── uploads/
├── frontend/               # Cliente Web
│   ├── index.html
│   ├── style.css
│   ├── js/
│   ├── pages/
│   └── assets/
└── _db/
    └── obraconnect_db.sql
```

### Verificar Conexão

```powershell
# Terminal 1: Backend
cd backend
npm start

# Output esperado:
# ╔═══════════════════════════════════════════════╗
# ║   🏗️  OBRACONNECT - SERVIDOR INICIADO        ║
# ║   Rodando em: http://localhost:3001              ║
# ║   Ambiente: development                  ║
# ╚═══════════════════════════════════════════════╝
# ✅ Conectado ao banco de dados com sucesso!

# Terminal 2: Frontend
cd frontend
# Abrir index.html no navegador
# Pode usar Live Server ou http-server
```

### Verificar API está respondendo

```javascript
// No console do navegador (F12)
fetch("http://localhost:3001/")
  .then(r => r.json())
  .then(d => console.log(d))

// Output esperado:
// {
//   "mensagem": "Bem-vindo ao ObraConnect Refatorado!",
//   "version": "1.0.0",
//   "endpoints": {...}
// }
```

---

## 🧪 Passo 2: Testar Fluxo Completo

### Teste 1: Registro e Login

```
1. Abrir https://localhost:3000
2. Clicar "Registrar"
3. Preencher formulário
4. Submeter
5. Banco MySQL registra usuário
6. Redirecionar para login
7. Fazer login
8. Backend retorna JWT
9. Frontend salva em localStorage
10. Redirecionar para index.html
11. ✅ Navbar mostra "Olá [Nome]"
```

### Teste 2: Criar Serviço

```
1. Estar logado como prestador
2. Se não for, clicar "Tornar Prestador" (seu perfil)
3. Clicar "Novo Serviço"
4. Preencher título, descrição, categoria, imagem
5. Submeter (envia FormData ao backend)
6. Multer salva imagem em /uploads
7. Backend cria registro no banco
8. Retorna sucesso e redireciona
9. ✅ Serviço aparece na lista com imagem
```

### Teste 3: Ver Detalhes e Avaliar

```
1. Clicar "Ver Detalhes" em um serviço
2. Página carrega dados (título, prestador, descrição)
3. Mostra avaliações em cards
4. Calcula nota média automaticamente
5. Se logado: formulário para avaliar
6. Preencher notas e comentário
7. Submeter
8. Backend registra no banco
9. Aparece nova avaliação na tela
10. Nota média atualiza automaticamente
11. ✅ Sistema funciona completo!
```

---

## 🔧 Passo 3: Depuração (Debug)

### Network Tab (F12 → Network)

```
1. Abrir DevTools (F12)
2. Ir para aba "Network"
3. Fazer uma ação (registro, login, criar serviço)
4. Ver requisições HTTP:
   
   POST /api/auth/registro
   Status: 201 ✅ ou 400 ❌
   Headers: Mostra _request headers
   Preview: Mostra response JSON

5. Se erro:
   - Ler mensagem de erro
   - Verificar que dados foram enviados
   - Verificar que backend validou
```

### Console.log Estratégico

**No Backend:**
```javascript
router.post("/login", async (req, res) => {
  console.log("📨 Requisição de login:", req.body);  // Ver dados
  console.log("🔍 User encontrado:", usuarios);       // Ver user
  console.log("✅ Token gerado:", token);             // Ver token
  // ...
});
```

**No Frontend:**
```javascript
async function realizarLogin(login, senha) {
  console.log("🔐 Fazendo login com:", login);
  const resultado = await fazerLogin(login, senha);
  console.log("📦 Resposta do backend:", resultado);
  console.log("💾 Token salvo:", localStorage.getItem("token"));
  // ...
}
```

### Erro Comum: CORS

```javascript
// ❌ Erro:
// Access to XMLHttpRequest at 'http://localhost:3001/...' 
// from origin 'http://localhost:5000' has been blocked

// ✅ Solução em backend/index.js:
app.use(cors({
  origin: "*",  // Permitir todos (depois restringir em produção)
  credentials: true
}));
```

---

## 🚀 Passo 4: Deploy - Renderizar Backend

### O que é Deploy?

```
Local (seu PC):
- Backend rodando em http://localhost:3001
- Frontend rodando em http://localhost:3000
- Você é o servidor

Production (internet):
- Backend rodando em cloud (AWS, Heroku, Render)
- Frontend rodando em CDN (Netlify, Vercel)
- Qualquer pessoa acessa
```

### Deploy Backend - Render (Grátis)

**1. Preparar Projeto**

```powershell
# Em backend/
# Garantir que index.js não tem hardcodes

# ❌ Ruim:
const PORT = 3001;
const DB_HOST = "localhost";

# ✅ Bom:
const PORT = process.env.PORT || 3001;
const DB_HOST = process.env.DB_HOST || "localhost";

# package.json deve ter:
{
  "engines": {
    "node": "18.x"  # Especificar versão
  }
}
```

**2. Criar Conta Render**

1. Ir em https://render.com
2. Clicar "Sign Up"
3. Criar conta com GitHub (recomendado)

**3. Fazer Deploy**

1. Clicar "New +"
2. Selecionar "Web Service"
3. Conectar repositório GitHub
4. Preencher:
   - **Name**: obraconnect-backend
   - **Runtime**: Node
   - **Branch**: main
   - **Build Command**: `npm install`
   - **Start Command**: `npm start`
5. Adicionar variáveis:
   - `DATABASE_URL=...` (string conexão)
   - `JWT_SECRET=sua_chave`
   - `JWT_EXPIRY=7d`
6. Clicar "Create Web Service"
7. Esperar ~2 min
8. ✅ Backend estava em https://seu-app.onrender.com

**4. Atualizar Frontend**

```javascript
// Em frontend/js/api.js

// ❌ Antes (localhost):
const API_BASE_URL = "http://localhost:3001/api";

// ✅ Depois (production):
const API_BASE_URL = "https://seu-app.onrender.com/api";

// Melhor: Usar variável de ambiente
const API_BASE_URL = process.env.REACT_APP_API_URL || "http://localhost:3001/api";
```

### Deploy Frontend - Netlify (Grátis)

**1. Preparar Projeto**

```powershell
# Criar arquivo netlify.toml na raiz de frontend/
[build]
  command = "ls"  # Frontend não precisa build (é HTML puro)
  publish = "."   # Publicar todos arquivos

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

**2. Fazer Deploy**

1. Ir em https://netlify.com
2. Clicar "Sign Up" (GitHub recomendado)
3. Clicar "Add new site" → "Import an existing project"
4. Selecionar repositório GitHub
5. Configurar:
   - **Branch**: main
   - **Publish directory**: . (raiz de frontend)
   - **Build command**: (deixar em branco)
6. Clicar "Deploy site"
7. Esperar ~1 min
8. ✅ Frontend está em https://seu-app.netlify.app

**3. Atualizar API_BASE_URL com URL do Render**

```javascript
// frontend/js/api.js
const API_BASE_URL = "https://seu-app.onrender.com/api";
```

---

## 🎯 Checklist Final de Deploy

**Backend (Render):**
- [ ] Código pushado para GitHub
- [ ] `.env.local` (.env não entra no Git)
- [ ] `package.json` tem `engines.node`
- [ ] Render conectado ao GitHub
- [ ] Variáveis de ambiente setadas
- [ ] Deploy bem-sucedido
- [ ] Testado em https://seu-app.onrender.com/

**Frontend (Netlify):**
- [ ] Código pushado para GitHub
- [ ] `api.js` com API_BASE_URL correto
- [ ] `netlify.toml` configurado
- [ ] Netlify conectado ao GitHub
- [ ] Deploy bem-sucedido
- [ ] Testado em https://seu-app.netlify.app

---

## 🔒 Passo 5: Segurança em Produção

### 1. Usar HTTPS (Obrigatório!)

```
❌ Sem HTTPS:
- Senhas viajam em texto plano
- Token JWT pode ser interceptado
- Qualquer um na rede vê seus dados

✅ Com HTTPS:
- Conexão criptografada
- Seguro contra homem no meio (MITM)
- Render/Netlify fornecem grátis
```

### 2. CORS Restritivo

```javascript
// ❌ Não usar em produção:
app.use(cors({ origin: "*" }));

// ✅ Restringir:
app.use(cors({
  origin: "https://seu-app.netlify.app",
  credentials: true,
  methods: ["GET", "POST", "PUT", "DELETE"],
  allowedHeaders: ["Content-Type", "Authorization"]
}));
```

### 3. Rate Limiting

```javascript
const rateLimit = require("express-rate-limit");

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutos
  max: 100,                   // 100 requisições
});

app.use("/api/", limiter);  // Aplicar em /api
```

### 4. Validar e Sanitizar

```javascript
const { body, validationResult } = require("express-validator");

router.post("/login",
  body("login").trim().isLength({ min: 3 }),
  body("senha").isLength({ min: 6 }),
  (req, res, next) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ erros: errors.array() });
    }
    next();
  }
);
```

---

## 📊 Monitorar em Produção

### Render Dashboard

1. Ir em https://render.com/dashboard
2. Selecionar seu serviço
3. Ver:
   - **Logs**: Erros e mensagens do servidor
   - **Metrics**: CPU, memória, banda
   - **Events**: Histórico de deploys

### Frontend (Netlify)

1. Ir em https://netlify.com/
2. Site → Analytics
3. Ver:
   - Requisições por dia
   - Erros de deploy
   - Tempo de construção

---

## 🎓 Resumo Arquitetura Final

```
┌─────────────────────────────────────────────────────────────┐
│                      INTERNET                                │
│                                                              │
│  ┌──────────────────────┐      ┌──────────────────────┐    │
│  │                      │      │                      │    │
│  │  Frontend            │      │  Backend             │    │
│  │  (Netlify)           │      │  (Render)            │    │
│  │                      │      │                      │    │
│  │  seu-app.netlify.app │◄────┤ seu-app.onrender.com │    │
│  │                      │      │                      │    │
│  │  ├── index.html      │      │  ├── index.js        │    │
│  │  ├── api.js ─────────┼──────┤► /api/auth           │    │
│  │  ├── auth.js         │      │  /api/servicos       │    │
│  │  └── servicos.js     │      │  /api/avaliacoes     │    │
│  │                      │      │                      │    │
│  └──────────────────────┘      └────────┬─────────────┘    │
│                                         │                   │
│                                         │ SQL               │
│                                         ▼                   │
│                                  ┌──────────────┐           │
│                                  │ MySQL (RDS)  │           │
│                                  │ obraconnect_ │           │
│                                  │ db           │           │
│                                  └──────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

---

## ✅ Sistema Completo!

Você agora tem:

✅ **Frontend**: Site responsivo em HTML/CSS/JS puro
✅ **Backend**: API REST com Express
✅ **Database**: MySQL com 4 tabelas e relationships
✅ **Autenticação**: JWT + bcryptjs
✅ **Upload**: Multer para imagens
✅ **CRUD**: Operações completas de serviços e avaliações
✅ **Deployment**: Frontend e Backend em produção
✅ **Segurança**: CORS, validação, hash de senhas

---

## 🚀 Próximos Passos (Melhorias Opcionais)

1. **Frontend Framework**: Migrar para React/Vue
2. **TypeScript**: Type safety no backend
3. **Testing**: Jest, Supertest para testes
4. **Docker**: Containerizar aplicação
5. **CI/CD**: Automatiazr testes e deploys com GitHub Actions
6. **Redis**: Cache para nota média de serviços
7. **Email**: Confirmação de registro, recuperar senha
8. **Paginação**: Listar 20 serviços por página ao invés de todos
9. **Search**: Buscar serviços por nome/categoria
10. **Admin Dashboard**: Gerenciar usuários e categorias

---

## 💡 Dica Final

**Versionamento:**
```bash
git add .
git commit -m "Deploy versão 1.0"
git tag -a v1.0 -m "Versão 1.0 - MVP completo"
git push origin main --tags
```

---

✅ **Parabéns!** Você completou o guia completo ObraConnect!

Se encontrar erros ou dúvidas, revise os passos anteriores ou consulte a documentação no folder `/ESTUDOS/ANALISE/`.

**Happy Coding! 🚀**
