# 🔧 ANÁLISE - FUNCIONALIDADES DO BACKEND

## 📋 Visão Geral

O backend do ObraConnect é uma **REST API** construída com **Node.js + Express** que gerencia:
- ✅ Autenticação de usuários (Registro, Login, JWT)
- ✅ Operações CRUD de serviços
- ✅ Sistema de avaliações
- ✅ Upload de imagens
- ✅ Validações de segurança

---

## 🚀 Estrutura do Backend

```
backend/
├── index.js                      # 🟢 Servidor Principal
├── package.json                  # Dependências
├── config/
│   ├── database.js              # 🟢 Pool MySQL
│   └── upload.js                # 🟢 Multer config
├── middlewares/
│   └── autenticacao.js          # 🟢 JWT verification
├── routes/
│   ├── authRoutes.js            # 🟢 Rotas Auth
│   ├── servicoRoutes.js         # 🟢 CRUD Serviços
│   └── avaliacaoRoutes.js       # 🟢 Avaliações
└── uploads/                      # Pasta imagens
```

---

## 🟢 Arquivo Principal: `index.js`

### Responsabilidades

```javascript
// 1. Importações e Setup
require("dotenv").config();        // Carregar variáveis de ambiente
const express = require("express");
const cors = require("cors");
const banco = require("./config/database");

// 2. Inicializar Express
const app = express();
const PORT = process.env.PORT || 3001;

// 3. Middlewares globais
app.use(cors(...));                // CORS
app.use(express.json());           // Parse JSON
app.use(express.urlencoded());     // Parse form data
app.use("/uploads", express.static(path.join(__dirname, "uploads")));

// 4. Rotas
app.use("/api/auth", authRoutes);
app.use("/api/servicos", servicoRoutes);
app.use("/api/avaliacoes", avaliacaoRoutes);

// 5. Tratamento de erros
app.use((req, res) => res.status(404).json(...));
app.use((erro, req, res, next) => res.status(500).json(...));

// 6. Iniciar servidor
app.listen(PORT, () => console.log(`Servidor rodando na porta ${PORT}`));
```

### Rotas Expostas

#### Teste

```
GET /
├─ Status: 200
└─ Response: {
    "mensagem": "Bem-vindo ao ObraConnect Refatorado!",
    "version": "1.0.0",
    "endpoints": { "auth": "/api/auth", ... }
  }

GET /teste-banco
├─ Status: 200 (connected) ou 500 (error)
└─ Response: { "mensagem": "✅ Banco conectado" }
```

#### Erro Padrão

```
GET /rota-nao-existe
├─ Status: 404
└─ Response: { "erro": "Rota não encontrada." }
```

---

## 🔐 Configuração do Banco: `config/database.js`

**O que faz**: Cria um pool de conexões com MySQL usando `mysql2/promise`

```javascript
const pool = mysql.createPool({
  host: process.env.DB_HOST || "localhost",
  user: process.env.DB_USER || "root",
  password: process.env.DB_PASSWORD || "",
  database: process.env.DB_NAME || "obraconnect_db",
  port: process.env.DB_PORT || 3306,
  waitForConnections: true,
  connectionLimit: 10,         // Max 10 conexões simultâneas
  queueLimit: 0,               // Fila ilimitada
});
```

### Por que usar Pool?

```
❌ Conexão Única
┌─────────────────────────┐
│ Requisição 1 → Esperar  │
│ Requisição 2 → Esperar  │
│ Requisição 3 → Esperar  │
└─────────────────────────┘
Causa: Gargalo, todas esperam em fila

✅ Pool de Conexões (10 conexões)
┌──────────────────────────────────────┐
│ Req 1 → Conexão 1 (simultâneo)        │
│ Req 2 → Conexão 2 (simultâneo)        │
│ ...                                   │
│ Req 10 → Conexão 10 (simultâneo)      │
│ Req 11 → Aguarda liberação            │
└──────────────────────────────────────┘
Causa: Melhor performance em produção
```

### Uso

```javascript
const [resultado] = await banco.query(
  "SELECT * FROM tabela WHERE id = ?",
  [id]
);
// Return: [rows, fields]
// Automático: Obtém conexão, executa, libera
```

---

## 📤 Configuração de Upload: `config/upload.js`

**O que faz**: Configura Multer para aceitar uploads de imagem

```javascript
const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    cb(null, uploadDir);  // Salva em /uploads
  },
  filename: function (req, file, cb) {
    // Nome único: nomeCampo-timestamp-random.ext
    cb(null, file.fieldname + "-" + uniqueSuffix + ext);
  },
});

// Filtro: Apenas imagens
const fileFilter = (req, file, cb) => {
  const extensoesPermitidas = /jpeg|jpg|png|gif/;
  const tiposPermitidos = /image\/jpeg|image\/png|image\/gif/;
  
  if (mimetype && extname) {
    cb(null, true);
  } else {
    cb(new Error("Apenas imagens permitidas!"));
  }
};

module.exports = multer({
  storage: storage,
  fileFilter: fileFilter,
  limits: { fileSize: 5 * 1024 * 1024 }  // Max 5MB
});
```

### Por que esta abordagem?

| Aspecto | Vantagem |
|---------|----------|
| **diskStorage** | Simples pra dev, funciona localmente |
| **Timestamp** | Impede conflict de nomes |
| **fileFilter** | Segurança: valida tipo arquivo |
| **Limite 5MB** | Evita abuso de storage |

### Limitações (melhorar em produção)

```
❌ Atual: Armazenar localmente em /uploads
  - Problema: Não escala (servidor único)
  - Nenhum backup automático
  
✅ Futuro: Usar AWS S3
  - Escala infinita
  - Backup automático
  - CDN incluído
```

---

## 🛡️ Middleware de Autenticação: `middlewares/autenticacao.js`

### 1. Função `verificarToken`

```javascript
function verificarToken(req, res, next) {
  // 1. Extrair token do header
  const token = req.headers.authorization?.split(" ")[1];
  // Formato: "Bearer JWTTOKEN"
  
  if (!token) return res.status(401).json({erro: "Token não fornecido"});
  
  try {
    // 2. Validar assinatura e decodificar
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    // 3. Adicionar dados decodificados à requisição
    req.usuario = decoded;
    
    // 4. Continuar
    next();
  } catch (erro) {
    if (erro.name === "TokenExpiredError") {
      return res.status(401).json({erro: "Token expirado"});
    }
    return res.status(403).json({erro: "Token inválido"});
  }
}
```

### Como funciona o JWT?

```
GERAR TOKEN (no login)
┌────────────────────────────────┐
│ Dados do usuário: {             │
│   id: 1,                        │
│   nome: "João",                 │
│   email: "joao@email.com",      │
│   tipo_usuario: "prestador"     │
│ }                               │
└──────────┬──────────────────────┘
           │
           ▼
┌────────────────────────────────┐
│ Assinado com JWT_SECRET         │
│ (senha privada do servidor)     │
│                                 │
│ Resultado:                       │
│ eyJhbGciOiJIUzI1NiIsInR5cCI     │
│ 6IkpXVCJ9.eyJpZCI6MSwibm9     │
│ tZSI6Ikpvw6NvIiwiZW1haWwiOi...│
└────────────────────────────────┘

VALIDAR TOKEN (em cada requisição protegida)
┌────────────────────────────────┐
│ Cliente envia:                   │
│ Authorization: Bearer TOKEN     │
└──────────┬──────────────────────┘
           │
           ▼
┌────────────────────────────────┐
│ Servidor verifica assinatura    │
│ com JWT_SECRET                  │
│                                 │
│ ✅ Se válido: Decodifica        │
│ ❌ Se inválido: Rejeita         │
│ 🕐 Se expirado: Pede novo       │
└────────────────────────────────┘
```

### 2. Função `verificarPrestador`

```javascript
function verificarPrestador(req, res, next) {
  if (req.usuario.tipo_usuario !== "prestador" && 
      req.usuario.tipo_usuario !== "admin") {
    return res.status(403).json({
      erro: "Apenas prestadores podem acessar"
    });
  }
  next();
}
```

**Uso**: Proteger rotas que somente prestadores podem usar
- POST /api/servicos (criar serviço)
- PUT /api/servicos/:id (editar serviço)
- DELETE /api/servicos/:id (deletar serviço)

---

## 🔑 Rotas de Autenticação: `routes/authRoutes.js`

### 1. POST /registro

```
Endpoint: POST http://localhost:3001/api/auth/registro

Request Body:
{
  "nome_usuario": "João Silva",
  "email": "joao@email.com",
  "senha": "senha123",
  "login": "joao_silva",
  "telefone": "11999999999"
}

Validações:
✅ Todos campos obrigatórios
✅ Email com formato válido
✅ Senha mín. 6 caracteres
✅ Email/login ainda não cadastrados

Fluxo:
1. Validar campos
2. Validar formato email (regex)
3. Validar comprimento senha
4. Verificar duplicatas (SELECT)
5. Criptografar senha com bcryptjs (genSalt + hash)
6. Inserir no DB (INSERT)
7. Retornar ID

Response (201 Created):
{
  "mensagem": "Usuário registrado com sucesso!",
  "id_usuario": 5,
  "usuario": "João Silva"
}
```

### Por que bcryptjs?

```
❌ Má prática: Armazenar senha em texto plano
  "senha": "123456" ← Se banco vazar, perdem acesso

✅ Boa prática: Hash com bcryptjs
  "senha": "$2a$10$N9qo8uLO..." ← Impossível recuperar senha original
  
Benefícios:
  - Salt aleatório: Mesma senha gera hash diferente
  - Lento propositál: Força bruta fica impraticável
  - Padrão indústria: Usado em grandes empresas
```

### 2. POST /login

```
Endpoint: POST http://localhost:3001/api/auth/login

Request Body:
{
  "login": "joao_silva",  // ou email
  "senha": "senha123"
}

Fluxo:
1. Validar campos
2. Buscar usuário por login ou email (SELECT)
3. Se não existe: Retornar 401
4. Comparar senha com bcryptjs.compare()
5. Se não match: Retornar 401
6. Gerar JWT com jwt.sign()
7. Retornar token

Response (200 OK):
{
  "mensagem": "Login realizado com sucesso!",
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "usuario": {
    "id": 1,
    "nome": "João Silva",
    "email": "joao@email.com",
    "tipo_usuario": "prestador"
  }
}
```

### JWT Expiração

```javascript
const token = jwt.sign(
  { id, nome, email, tipo_usuario },
  process.env.JWT_SECRET,
  { expiresIn: process.env.JWT_EXPIRY }  // Ex: "7d"
);
```

**Por quê expiração?**
```
Se token nunca expirasse:
- Token vazado = acesso permanente
- Logout seria inútil (token ainda válido)

Com expiração (7 dias):
- Token vazado = acesso limitado
- Força user logar periodicamente
- Aumenta segurança
```

### 3. GET /perfil

```
Endpoint: GET http://localhost:3001/api/auth/perfil
Requer: Authorization: Bearer TOKEN

Fluxo:
1. Verificar token (middleware)
2. ID do usuário vem do token (req.usuario.id)
3. Buscar dados atualizados do DB (SELECT)
4. Não retorna senha

Response (200 OK):
{
  "id": 1,
  "nome_usuario": "João Silva",
  "email": "joao@email.com",
  "login": "joao_silva",
  "telefone": "11999999999",
  "tipo_usuario": "prestador",
  "data_cadastro": "2026-01-22T19:08:35.000Z"
}
```

### 4. PUT /tornar-prestador

```
Endpoint: PUT http://localhost:3001/api/auth/tornar-prestador
Requer: Authorization: Bearer TOKEN

Fluxo:
1. Verificar token
2. Verificar se já é prestador
3. UPDATE tipo_usuario = 'prestador'
4. Retornar mensagem

Response (200 OK):
{
  "mensagem": "Você agora é um prestador!"
}
```

---

## 🛠️ Rotas de Serviços: `routes/servicoRoutes.js`

### 1. GET / (Listar todos)

```
Endpoint: GET http://localhost:3001/api/servicos
Autenticação: NÃO REQUERIDA (público)

Query Complexa (com JOINs e agregação):
SELECT 
  s.*,
  u.nome_usuario,
  u.email,
  AVG((a.nota_preco + a.nota_tempo_execucao + a.nota_higiene + a.nota_educacao) / 4) as nota_media,
  COUNT(a.id) as total_avaliacoes
FROM oc__tb_servico s
JOIN oc__tb_usuario u ON s.id_usuario = u.id
LEFT JOIN oc__tb_avaliacao a ON s.id = a.id_servico
GROUP BY s.id
ORDER BY s.data_cadastro DESC

Response (200 OK):
[
  {
    "id": 1,
    "id_usuario": 1,
    "titulo": "Encanamento",
    "desc_servico": "...",
    "id_categoria": 4,
    "imagem_url": "http://...",
    "data_cadastro": "2026-01-25T14:18:21.000Z",
    "nome_usuario": "João Silva",
    "email": "joao@email.com",
    "nota_media": 4.75,
    "total_avaliacoes": 4
  },
  ...
]
```

**Por que não usar SELECT *?**
```
❌ SELECT * FROM oc__tb_servico
  - Retorna todos os campos sem processamento
  - Sem dados do prestador
  - Sem nota média
  
✅ SELECT s.*, u.nome_usuario, AVG(...) as nota_media
  - Dados completos
  - Nota média calculada no DB (mais eficiente)
  - Tudo que frontend precisa
```

### 2. GET /:id (Ver detalhes)

```
Endpoint: GET http://localhost:3001/api/servicos/1
Autenticação: NÃO REQUERIDA (público)

Fluxo:
1. Pegar ID da URL
2. Mesma query anterior mas WHERE s.id = ?
3. Se não encontra: 404
4. Se encontra: Retornar com nota_media parseada

Response (200 OK):
{
  "id": 1,
  "id_usuario": 1,
  "titulo": "Encanamento em Geral",
  "desc_servico": "Conserto de vazamentos...",
  "id_categoria": 4,
  "imagem_url": "http://localhost:3001/uploads/imagem-1234567890.jpg",
  "data_cadastro": "2026-01-25T14:18:21.000Z",
  "nome_usuario": "João Silva",
  "email": "joao@email.com",
  "telefone": "11999999999",
  "nota_media": 4.75,
  "total_avaliacoes": 4
}
```

### 3. GET /meus/servicos (Meus serviços)

```
Endpoint: GET http://localhost:3001/api/servicos/meus/servicos
Autenticação: REQUERIDA (verificarToken)
Nota: DEVE VIR ANTES DE /:id na rota para não confundir

Fluxo:
1. Verificar token
2. ID do usuário do token (req.usuario.id)
3. SELECT * WHERE id_usuario = ?
4. Retornar meus serviços

Response (200 OK):
[
  {
    "id": 1,
    "id_usuario": 1,
    "titulo": "Encanamento",
    ...
  }
]
```

**Por que /meus/servicos ANTES de /:id?**

```
Se ordem fosse:
GET /:id → trata "meus" como ID
GET /meus/servicos → nunca executa

Ordem correta:
GET /meus/servicos → específico executa
GET /:id → genérico executa
```

### 4. POST / (Criar serviço)

```
Endpoint: POST http://localhost:3001/api/servicos
Autenticação: REQUERIDA + verificarPrestador
Content-Type: multipart/form-data (upload)

Request:
{
  "titulo": "Encanamento",
  "descricao": "Conserto de vazamentos...",
  "id_categoria": 4,
  "imagem": <arquivo>  // Arquivo de imagem
}

Validações:
✅ Título obrigatório
✅ Descrição obrigatória
✅ Arquivo é imagem (Multer valida)
✅ Apenas prestadores

Fluxo:
1. Verificar token → req.usuario.id
2. Verificar if prestador
3. Multer salva arquivo em /uploads
4. Gerar URL: http://localhost:3001/uploads/image-timestamp.jpg
5. INSERT no DB
6. Retornar ID e URL

Response (201 Created):
{
  "mensagem": "Serviço criado com sucesso!",
  "id_servico": 5,
  "imagem": "http://localhost:3001/uploads/imagem-1234567890.jpg"
}
```

### 5. PUT /:id (Editar)

```
Endpoint: PUT http://localhost:3001/api/servicos/1
Autenticação: REQUERIDA + verificarPrestador

Request:
{
  "titulo": "Novo título",
  "descricao": "Nova descrição"
}

Validações:
✅ Verificar se serviço pertence ao usuário
✅ Apenas o criador pode editar

Fluxo:
1. Verificar token
2. Verificar if prestador
3. SELECT para verificar proprietário
4. Se não é dono: 403 Forbidden
5. UPDATE campos alterados
6. Retornar mensagem

Response (200 OK):
{
  "mensagem": "Serviço atualizado!"
}
```

### 6. DELETE /:id (Deletar)

```
Endpoint: DELETE http://localhost:3001/api/servicos/1
Autenticação: REQUERIDA + verificarPrestador

Fluxo:
1. Verificar token
2. Verificar if prestador
3. Verificar proprietário
4. DELETE FROM oc__tb_servico WHERE id
5. DELETE avaliações associadas (CASCADE teórico)
6. Retornar mensagem

Response (200 OK):
{
  "mensagem": "Serviço deletado com sucesso!"
}
```

---

## ⭐ Rotas de Avaliações: `routes/avaliacaoRoutes.js`

### 1. GET /servico/:id (Listar avaliações)

```
Endpoint: GET http://localhost:3001/api/avaliacoes/servico/1
Autenticação: NÃO REQUERIDA

Fluxo:
1. Pegar ID do serviço
2. SELECT com JOIN de usuário
3. Se houver avaliações: calcular médias
4. Retornar com estatísticas

Response (200 OK):
{
  "total_avaliacoes": 4,
  "medias": {
    "preco": 4.8,
    "tempo": 4.5,
    "higiene": 4.8,
    "educacao": 4.8,
    "geral": 4.73
  },
  "avaliacoes": [
    {
      "id": 1,
      "id_servico": 1,
      "id_usuario": 2,
      "nota_preco": 5,
      "nota_tempo_execucao": 4,
      "nota_higiene": 5,
      "nota_educacao": 5,
      "comentario": "Excelente profissional!",
      "data_avaliacao": "2026-01-22T19:08:35.000Z",
      "nome_usuario": "Maria Silva"
    }
  ]
}
```

### 2. POST / (Criar avaliação)

```
Endpoint: POST http://localhost:3001/api/avaliacoes
Autenticação: REQUERIDA

Request:
{
  "id_servico": 1,
  "nota_preco": 5,
  "nota_tempo_execucao": 4,
  "nota_higiene": 5,
  "nota_educacao": 5,
  "comentario": "Excelente!"
}

Validações:
✅ Notas entre 1-5 (frontend + backend)
✅ Serviço existe
✅ Não avaliar próprio serviço
✅ Já avaliou? (opcional verificar)

Fluxo:
1. Verificar token → ID do avaliador
2. Validar notas (1-5)
3. Verificar se criador não avalia próprio
4. INSERT no DB
5. Retornar novo total

Response (201 Created):
{
  "mensagem": "Avaliação registrada!",
  "id_avaliacao": 5
}
```

### 3. DELETE /:id (Deletar avaliação)

```
Endpoint: DELETE http://localhost:3001/api/avaliacoes/5
Autenticação: REQUERIDA

Fluxo:
1. Verificar token
2. SELECT para verificar proprietário da avaliação
3. DELETE WHERE id

Response (200 OK):
{
  "mensagem": "Avaliação deletada!"
}
```

---

## 🎯 Resumo de Endpoints

| Método | Endpoint | Auth | Função |
|--------|----------|------|--------|
| GET | `/` | ❌ | Status servidor |
| GET | `/teste-banco` | ❌ | Teste conexão |
| **AUTH** |
| POST | `/api/auth/registro` | ❌ | Novo usuário |
| POST | `/api/auth/login` | ❌ | Login |
| GET | `/api/auth/perfil` | ✅ | Ver perfil |
| PUT | `/api/auth/tornar-prestador` | ✅ | Virar prestador |
| **SERVIÇOS** |
| GET | `/api/servicos` | ❌ | Listar todos |
| GET | `/api/servicos/:id` | ❌ | Ver detalhes |
| GET | `/api/servicos/meus/servicos` | ✅ | Meus serviços |
| POST | `/api/servicos` | ✅+prestador | Criar |
| PUT | `/api/servicos/:id` | ✅+prestador | Editar |
| DELETE | `/api/servicos/:id` | ✅+prestador | Deletar |
| **AVALIAÇÕES** |
| GET | `/api/avaliacoes/servico/:id` | ❌ | Listar avaliações |
| GET | `/api/avaliacoes/meu-historico` | ✅ | Minhas avaliações |
| POST | `/api/avaliacoes` | ✅ | Criar avaliação |
| DELETE | `/api/avaliacoes/:id` | ✅ | Deletar avaliação |

---

## 🔍 Padrões de Código Usado

### 1. Try-Catch para Tratamento de Erro
```javascript
try {
  const [resultado] = await banco.query(...);
  res.status(200).json(resultado);
} catch (erro) {
  console.error("Erro:", erro);
  res.status(500).json({erro: "Erro ao processar"});
}
```

### 2. Validação de Campos
```javascript
if (!campo) return res.status(400).json({erro: "Campo obrigatório"});
```

### 3. Query Parametrizada (SQL Injection Prevention)
```javascript
// ✅ Seguro: Valor vira ?
banco.query("SELECT * FROM tabela WHERE id = ?", [id]);

// ❌ Inseguro: Concatenação
banco.query(`SELECT * FROM tabela WHERE id = ${id}`);
```

### 4. Middlewares em Sequência
```javascript
router.post(
  "/",
  verificarToken,              // 1º: Validar JWT
  verificarPrestador,          // 2º: Validar tipo usuário
  upload.single("imagem"),     // 3º: Upload de arquivo
  async (req, res) => {...}    // 4º: Lógica principal
);
```

---

Este é o núcleo funcional do ObraConnect! Todas as operações passam por essas rotas e middlewares.
