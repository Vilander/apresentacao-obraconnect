# 📚 GUIA PASSO A PASSO - 02: ESTRUTURA E CONFIGURAÇÃO DO PROJETO

## 🎯 Objetivo

Criar a estrutura base do projeto backend com configurações iniciais

---

## 📁 Passo 1: Criar Estrutura de Pastas

Após completar o Passo 1 (ambiente), sua estrutura deve ser:

```
backend/
├── package.json
├── package-lock.json
├── .env
├── .gitignore
├── node_modules/
```

Agora criar as pastas adicionais:

```powershell
# Em backend/

# Criar pastas de configuração
mkdir config
mkdir middlewares
mkdir routes

# Criar pasta de uploads
mkdir uploads

# Verificar estrutura
tree

# Output esperado:
# backend
# ├── config/
# ├── middlewares/
# ├── routes/
# ├── uploads/
# ├── package.json
# ├── .env
# └── node_modules/
```

---

## 🟢 Passo 2: Criar Arquivo Principal `index.js`

**O que faz?** Inicializa o servidor Express e registra rotas

**Localização:** `backend/index.js`

```javascript
/**
 * SERVIDOR PRINCIPAL - ObraConnect Refatorado
 * Marketplace de serviços de construção
 *
 * Tecnologias: Node.js + Express + MySQL
 * Frontend: HTML + Bootstrap + JavaScript Vanilla
 */

// 1. Importar variáveis de ambiente
require("dotenv").config();

// 2. Importar dependências
const express = require("express");
const cors = require("cors");
const path = require("path");
const banco = require("./config/database");

// 3. Importar rotas (criar depois)
// const authRoutes = require("./routes/authRoutes");
// const servicoRoutes = require("./routes/servicoRoutes");
// const avaliacaoRoutes = require("./routes/avaliacaoRoutes");

// 4. Inicializar Express
const app = express();
const PORT = process.env.PORT || 3001;

// ===============================================
// MIDDLEWARES GLOBAIS (executam em TODAS requisições)
// ===============================================

// CORS - Permitir requisições do frontend
app.use(
  cors({
    origin: "*",  // Permitir qualquer origem (depois restringir em prod)
    methods: ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
    credentials: true,
  }),
);

// Body Parser - Parsear JSON no corpo das requisições
app.use(express.json());

// URL Encoded - Parsear form data
app.use(express.urlencoded({ extended: true }));

// Servir arquivos estáticos (imagens dos serviços)
app.use("/uploads", express.static(path.join(__dirname, "uploads")));

// ===============================================
// ROTAS DE TESTE
// ===============================================

// Rota raiz
app.get("/", (req, res) => {
  res.status(200).json({
    mensagem: "Bem-vindo ao ObraConnect Refatorado!",
    version: "1.0.0",
    endpoints: {
      auth: "/api/auth",
      servicos: "/api/servicos",
      avaliacoes: "/api/avaliacoes",
    },
  });
});

// Teste de conexão com banco
app.get("/teste-banco", async (req, res) => {
  try {
    const [resultado] = await banco.query("SELECT 1 as teste");
    res.status(200).json({
      mensagem: "✅ Banco de dados conectado com sucesso!",
      teste: resultado,
    });
  } catch (erro) {
    console.error("Erro ao testar banco:", erro);
    res.status(500).json({
      erro: "❌ Erro ao conectar no banco de dados",
      detalhes: erro.message,
    });
  }
});

// ===============================================
// REGISTRAR ROTAS (descomentar quando criar)
// ===============================================

// app.use("/api/auth", authRoutes);
// app.use("/api/servicos", servicoRoutes);
// app.use("/api/avaliacoes", avaliacaoRoutes);

// ===============================================
// TRATAMENTO DE ERROS
// ===============================================

// 404 - Rota não encontrada
app.use((req, res) => {
  res.status(404).json({
    erro: "Rota não encontrada.",
    metodo: req.method,
    path: req.path,
  });
});

// Erro geral (middleware com 4 parâmetros sempre é error handler)
app.use((erro, req, res, next) => {
  console.error("❌ Erro no servidor:", erro);
  
  res.status(500).json({
    erro: "Erro interno do servidor.",
    detalhes: process.env.NODE_ENV === "development" ? erro.message : undefined,
    stack: process.env.NODE_ENV === "development" ? erro.stack : undefined,
  });
});

// ===============================================
// INICIAR SERVIDOR
// ===============================================

app.listen(PORT, () => {
  console.log(`
  ╔═══════════════════════════════════════════════╗
  ║   🏗️  OBRACONNECT - SERVIDOR INICIADO        ║
  ║   Rodando em: http://localhost:${PORT}              ║
  ║   Ambiente: ${process.env.NODE_ENV || "development"}                  ║
  ╚═══════════════════════════════════════════════╝
  `);
});
```

**Por quê essa estrutura?**

```
1. require("dotenv").config() PRIMEIRO
   └─ Carrega variáveis de ambiente antes de usar

2. Importar dependências ANTES de criar app
   └─ Garante tudo disponível quando precisar

3. Middlewares ANTES de rotas
   └─ Middlewares executam em TODAS requisições
   └─ Se não estivessem antes, algumas rotas ignorariam

4. Rotas DEPOIS de middlewares
   └─ Herdam configuração dos middlewares

5. Tratamento de erro POR ÚLTIMO
   └─ Error handler só executa se houve erro
   └─ Deve ser último porque captura erros das rotas
```

---

## 🟢 Passo 3: Criar Configuração do Banco `config/database.js`

**O que faz?** Conectar ao MySQL e criar pool de conexões

**Localização:** `backend/config/database.js`

```javascript
/**
 * Configuração de Conexão com MySQL
 * Usa mysql2/promise para operações assíncronas
 */

require("dotenv").config();
const mysql = require("mysql2/promise");

// ===============================================
// CRIAR POOL DE CONEXÕES
// ===============================================

/**
 * Pool: Mantém N conexões abertas
 * - Quando requisição chega: pega uma conexão do pool
 * - Executa query
 * - Libera conexão de volta para pool
 * 
 * Benefício: Não precisa criar nova conexão a cada requisição
 * Performance: 100x mais rápido que conexão única
 */

const pool = mysql.createPool({
  host: process.env.DB_HOST || "localhost",
  user: process.env.DB_USER || "root",
  password: process.env.DB_PASSWORD || "",
  database: process.env.DB_NAME || "obraconnect_db",
  port: process.env.DB_PORT || 3306,
  
  // Configurações de pool
  waitForConnections: true,      // Esperar se não houver conexão disponível
  connectionLimit: 10,           // Máximo 10 conexões simultâneas
  queueLimit: 0,                 // Fila ilimitada de espera
});

// ===============================================
// TESTAR CONEXÃO AO INICIAR
// ===============================================

pool
  .getConnection()
  .then((connection) => {
    console.log("✅ Conectado ao banco de dados com sucesso!");
    connection.release();  // Devolver conexão ao pool
  })
  .catch((erro) => {
    console.error("❌ Erro ao conectar no banco de dados:", erro.message);
    
    // Se erro ao conectar, parar servidor (não pode rodar sem banco)
    process.exit(1);
  });

// ===============================================
// EXPORTAR POOL
// ===============================================

/**
 * Uso:
 * 
 * const banco = require("./config/database");
 * 
 * // Fazer query
 * const [resultado] = await banco.query(
 *   "SELECT * FROM oc__tb_usuario WHERE id = ?",
 *   [id]
 * );
 * 
 * // Resultado: [rows, fields]
 * // Pool cuida automaticamente: obtém conexão → executa → libera
 */

module.exports = pool;
```

**Por quê pool e não conexão única?**

```
CONEXÃO ÚNICA (❌ Ruim para produção)
┌──────────────────────────┐
│ Requisição 1             │
└─────────────────────────→│
                            │ Executa query
                            │ (outros esperam)
┌──────────────────────────┐│
│ Requisição 2             ││ Fila
└─────────────────────────→││
┌──────────────────────────┐││
│ Requisição 3             │││ Espera...
└──────────────────────────┘
Gargalo: Requisições esperam em fila


POOL DE CONEXÕES (✅ Bom para produção)
┌──────────────────────────┐
│ Requisição 1 → [Conexão 1]  Executa simultaneamente
│ Requisição 2 → [Conexão 2]  
│ Requisição 3 → [Conexão 3]
│ ...
│ Requisição 10 → [Conexão 10]
│ Requisição 11 → Aguarda liberação (fast)
└──────────────────────────┘
Rápido: Conexões reutilizadas
```

---

## 🟢 Passo 4: Criar Configuração de Upload `config/upload.js`

**O que faz?** Configurar Multer para aceitar uploads de imagem

**Localização:** `backend/config/upload.js`

```javascript
/**
 * Configuração do Multer para upload de imagens
 * Salva as fotos na pasta /uploads
 */

const multer = require("multer");
const path = require("path");
const fs = require("fs");

// ===============================================
// CRIAR PASTA UPLOADS SE NÃO EXISTIR
// ===============================================

const uploadDir = path.join(__dirname, "../uploads");

if (!fs.existsSync(uploadDir)) {
  fs.mkdirSync(uploadDir, { recursive: true });
  console.log("📁 Pasta /uploads criada");
}

// ===============================================
// CONFIGURAR STORAGE
// ===============================================

/**
 * diskStorage: Salvar arquivo no disco (local)
 * 
 * destination: Define em qual pasta salvar
 * filename: Define nome do arquivo
 */

const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    // Salvar em pasta /uploads
    cb(null, uploadDir);
  },
  
  filename: function (req, file, cb) {
    /**
     * Gerar nome único para evitar conflitos
     * 
     * Exemplo:
     * Original: minha-imagem.jpg
     * Salvo: imagem-1232131223-123123123.jpg
     * 
     * Date.now(): Timestamp (milissegundos desde 1970)
     * Math.random(): Número aleatório
     * 
     * Garante: Nunca haverá dois arquivos iguais
     */
    
    const uniqueSuffix = Date.now() + "-" + Math.round(Math.random() * 1e9);
    const extensao = path.extname(file.originalname);  // .jpg, .png
    const nomeBase = path.basename(file.originalname, extensao);  // nome sem extensão
    
    cb(null, nomeBase + "-" + uniqueSuffix + extensao);
  },
});

// ===============================================
// FILTRO DE ARQUIVOS
// ===============================================

/**
 * Validar tipo de arquivo
 * - Apenas imagens
 * - Extensões: jpeg, jpg, png, gif
 * - MIME types: image/jpeg, image/png, etc
 */

const fileFilter = (req, file, cb) => {
  // Extensões permitidas
  const extensoesPermitidas = /jpeg|jpg|png|gif/;
  const extensao = extensoesPermitidas.test(
    path.extname(file.originalname).toLowerCase()
  );
  
  // MIME types permitidos
  const tiposPermitidos = /image\/jpeg|image\/png|image\/gif/;
  const mimetype = tiposPermitidos.test(file.mimetype);
  
  // Se ambos válidos: aceitar
  if (mimetype && extensao) {
    return cb(null, true);
  } else {
    // Se inválido: rejeitar com erro
    cb(new Error("Apenas imagens (jpeg, jpg, png, gif) são permitidas!"));
  }
};

// ===============================================
// CRIAR INSTÂNCIA MULTER
// ===============================================

const upload = multer({
  storage: storage,
  fileFilter: fileFilter,
  limits: {
    fileSize: parseInt(process.env.MAX_FILE_SIZE) || 5 * 1024 * 1024,  // 5MB padrão
  },
});

// ===============================================
// EXPORTAR
// ===============================================

/**
 * Uso:
 * 
 * const upload = require("./config/upload");
 * 
 * router.post(
 *   "/criar-servico",
 *   upload.single("imagem"),  // Aceitar 1 arquivo no campo "imagem"
 *   async (req, res) => {
 *     // req.file contém: originalname, fieldname, filename, path, size, mimetype
 *     const arquivo = req.file;
 *     
 *     if (!arquivo) {
 *       return res.status(400).json({erro: "Arquivo não enviado"});
 *     }
 *     
 *     // req.file.path: "uploads/imagem-123123.jpg"
 *     // req.file.filename: "imagem-123123.jpg"
 *     
 *     res.json({url: `http://localhost:3001/uploads/${req.file.filename}`});
 *   }
 * );
 */

module.exports = upload;
```

**Por quê essa configuração?**

```
❌ SEM filtro
- User envia arquivo.exe
- Multer salva
- Problema: Malware no servidor!

✅ COM filtro
- User envia arquivo.exe
- Multer valida: "Não é imagem!"
- Rejeita antes de salvar

❌ SEM nome único
- Dois users enviam mesma imagem.jpg
- Segundo sobrescreve primeiro
- Problema: Imagem perdida!

✅ COM timestamp
- User 1 envia imagem.jpg → salvo como imagem-1234567-123.jpg
- User 2 envia imagem.jpg → salvo como imagem-1234568-456.jpg
- Problema resolvido!
```

---

## ✅ Checklist de Conclusão

- [ ] Pastas criadas: config/, middlewares/, routes/, uploads/
- [ ] `backend/index.js` criado com estrutura completa
- [ ] `backend/config/database.js` criado com pool MySQL
- [ ] `backend/config/upload.js` criado com Multer
- [ ] `.env` preenchido com variáveis corretas

---

## 🧪 Testar Servidor

```powershell
# Em backend/
npm start

# Saída esperada:
# ╔═══════════════════════════════════════════════╗
# ║   🏗️  OBRACONNECT - SERVIDOR INICIADO        ║
# ║   Rodando em: http://localhost:3001              ║
# ║   Ambiente: development                  ║
# ╚═══════════════════════════════════════════════╝
# ✅ Conectado ao banco de dados com sucesso!
```

Testar rotas de teste:

```
GET http://localhost:3001/
Response: {"mensagem": "Bem-vindo ao ObraConnect...", ...}

GET http://localhost:3001/teste-banco
Response: {"mensagem": "✅ Banco de dados conectado...", ...}
```

---

## 💡 Estrutura Completa agora:

```
backend/
├── index.js                    # ✅ Criado
├── config/
│   ├── database.js            # ✅ Criado
│   └── upload.js              # ✅ Criado
├── middlewares/
│   └── autenticacao.js        # Criar no próximo passo
├── routes/
│   ├── authRoutes.js          # Criar no próximo passo
│   ├── servicoRoutes.js       # Criar no próximo passo
│   └── avaliacaoRoutes.js     # Criar no próximo passo
├── uploads/                    # Pasta vazia (receberá imagens)
├── package.json               # ✅ Criado
├── .env                        # ✅ Criado
└── node_modules/              # ✅ Criado
```

---

✅ **Servidor base está rodando!** Próximo passo: Criar banco de dados e tabelas.
