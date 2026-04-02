# Guia passo a passo para replicar obraConnect do zero

> Este documento ensina, em detalhes total, como recriar o projeto obraConnect (frontend, backend, database) a partir de uma estrutura vazia.

## 1. Visão geral do projeto

- **Frontend**: páginas HTML, CSS e JavaScript rodando no navegador.
- **Backend**: servidor Node.js com Express e rotas REST para autenticação, serviços e avaliações.
- **Banco de dados**: MySQL para persistir usuários, serviços, avaliações.

O fluxo da aplicação:
1. Usuário usa formulário no navegador.
2. Frontend faz requisição HTTP para o backend (`fetch`).
3. Backend executa lógica, acessa MySQL e retorna JSON.
4. Frontend atualiza interface com os dados.

---

## 2. Estrutura de pastas que você vai criar

Na raiz do repositório, crie as seguintes pastas e arquivos:

- `/frontend/`
  - `index.html`
  - `style.css`
  - `/js/`
    - `api.js`
    - `auth.js`
    - `servicos.js`
  - `/pages/`
    - `login.html`
    - `registro.html`
    - `meu-perfil.html`
    - `cadastrar-servico.html`
    - `detalhes-servico.html`

- `/backend/`
  - `index.js`
  - `package.json` (gerado pelo npm)
  - `.env` (local, não versionar)
  - `/config/`
    - `database.js`
    - `upload.js`
  - `/middlewares/`
    - `autenticacao.js`
  - `/routes/`
    - `authRoutes.js`
    - `servicoRoutes.js`
    - `avaliacaoRoutes.js`
    - `testesRoutes.js`

- `/_db/`
  - `obraconnect_db.sql`

- `/ESTUDOS/PASSO_A_PASSO/` (este arquivo)

---

## 3. Configurar e criar banco de dados (MySQL)

1. Instale o MySQL: https://dev.mysql.com/downloads/mysql/
2. Inicie serviço MySQL (Windows: MySQL80 no services).
3. Abra terminal e autentique:

```bash
mysql -u root -p
```

4. Crie banco:

```sql
CREATE DATABASE IF NOT EXISTS obraconnect;
USE obraconnect;
```

5. Crie tabelas (
crie o arquivo `_db/obraconnect_db.sql` com este conteúdo):

```sql
-- criar tabela de usuários
CREATE TABLE users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  nome VARCHAR(100) NOT NULL,
  email VARCHAR(150) NOT NULL UNIQUE,
  senha VARCHAR(255) NOT NULL,
  criado_em DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- criar tabela de serviços
CREATE TABLE servicos (
  id INT AUTO_INCREMENT PRIMARY KEY,
  titulo VARCHAR(150) NOT NULL,
  descricao TEXT NOT NULL,
  preco DECIMAL(10,2) NOT NULL,
  imagem VARCHAR(255),
  usuario_id INT NOT NULL,
  criado_em DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (usuario_id) REFERENCES users(id) ON DELETE CASCADE
);

-- criar tabela de avaliacoes
CREATE TABLE avaliacoes (
  id INT AUTO_INCREMENT PRIMARY KEY,
  servico_id INT NOT NULL,
  usuario_id INT NOT NULL,
  nota INT NOT NULL,
  comentario TEXT,
  criado_em DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (servico_id) REFERENCES servicos(id) ON DELETE CASCADE,
  FOREIGN KEY (usuario_id) REFERENCES users(id) ON DELETE CASCADE
);
```

6. Importar diretamente, se quiser:

```bash
mysql -u root -p obraconnect < _db/obraconnect_db.sql
```

Explicação linha a linha:
- `CREATE TABLE`: instrução do MySQL para criar uma tabela.
- `id INT AUTO_INCREMENT PRIMARY KEY`: auto-incrementa e garante valor único.
- `VARCHAR(100)`, `TEXT`, `DECIMAL(10,2)` definem tipo e tamanho do campo.
- `FOREIGN KEY` garante relação com outras tabelas.
- `ON DELETE CASCADE` faz apagar registros dependentes quando pai é removido.

---

## 4. Backend Node.js (passo a passo)

### 4.1. Inicializar projeto backend

1. Abra terminal em `backend/`.
2. Execute `npm init -y`.
3. Execute:

```bash
npm install express mysql2 dotenv bcryptjs jsonwebtoken cors multer
npm install --save-dev nodemon
```

- `express`: framework HTTP do Node.js, cria rotas de API.
- `mysql2`: driver do MySQL com suporte a promessas.
- `dotenv`: lê arquivo `.env` em `process.env`.
- `bcryptjs`: encripta senhas.
- `jsonwebtoken`: gera tokens JWT.
- `cors`: libera acesso de frontend remoto.
- `multer`: trata uploads de arquivos (imagens).
- `nodemon`: reinicia servidor automaticamente durante desenvolvimento.

### 4.2. Criar `.env`

Local: `/backend/.env`

```ini
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=sua_senha_aqui
DB_NAME=obraconnect
JWT_SECRET=sua_chave_super_secreta
PORT=3001
NODE_ENV=development
```

Explanação:
- Cada linha é `NOME=VALOR`.
- `DB_...` são credenciais do banco.
- `JWT_SECRET` é segredo para assinar tokens; não deve estar no Git.
- `PORT` porta do servidor.

### 4.3. Crie `backend/index.js`

Crie o arquivo com este código completo:

```javascript
require("dotenv").config();
const express = require("express");
const cors = require("cors");
const path = require("path");
const banco = require("./config/database");

// Importar rotas
const authRoutes = require("./routes/authRoutes");
const servicoRoutes = require("./routes/servicoRoutes");
const avaliacaoRoutes = require("./routes/avaliacaoRoutes");
const testesRoutes = require("./routes/testesRoutes");

// Inicializar Express
const app = express();
const PORT = process.env.PORT || 3001;

// CORS - permite requisições de qualquer origem no desenvolvimento
app.use(
  cors({
    origin: "*",
    methods: ["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
    credentials: true,
  }),
);

// Body parser JSON
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Servir arquivos estáticos (imagens)
app.use("/uploads", express.static(path.join(__dirname, "uploads")));

// Rotas
app.use("/api/auth", authRoutes);
app.use("/api/servicos", servicoRoutes);
app.use("/api/avaliacoes", avaliacaoRoutes);
app.use("/api/testes", testesRoutes);

// Rotas não encontradas (404)
app.use((req, res) => {
  res.status(404).json({ erro: "Rota não encontrada." });
});

// Tratamento de erros geral
app.use((erro, req, res, next) => {
  console.error("Erro no servidor:", erro);
  res.status(500).json({
    erro: "Erro interno do servidor.",
    detalhes: process.env.NODE_ENV === "development" ? erro.message : undefined,
  });
});

// Iniciar servidor
app.listen(PORT, () => {
  console.log(`Servidor rodando em http://localhost:${PORT}`);
});

process.on("SIGTERM", () => {
  console.log("Servidor encerrando...");
  process.exit(0);
});
```

Explicação passo a passo (cada linha):
- `require("dotenv").config();` carrega variáveis de ambiente de `.env` para `process.env`.
- `const express = require("express");` importa o framework Express.
- `const cors = require("cors");` importa middleware para CORS (cross-origin resource sharing).
- `const path = require("path");` facilita caminhos de arquivos no sistema.
- `const banco = require("./config/database");` importa conexão com MySQL para garantir que arquivos de rota possam usar.
- `app.use(cors(...))`: configura e libera chamadas do frontend.
- `app.use(express.json())`: permite que o corpo JSON de requisições seja lido em `req.body`.
- `app.use(express.urlencoded({ extended: true }))`: permite processar formulários `x-www-form-urlencoded`.
- `app.use("/uploads", express.static(...))`: disponibiliza pasta de uploads como rota pública `/uploads/`.
- `app.use("/api/auth", authRoutes);`: monta rotas de autenticação em `/api/auth`.
- `app.use((req,res)=>...)`: middleware final para 404.
- `app.use((erro, req, res, next) => ...)`: middleware de tratamento de erro do Express.
- `app.listen(PORT, ...)`: inicia servidor.
- `process.on("SIGTERM" ...)`: fecha processo com mensagem quando recepciona sinal de término.

### 4.4. Crie `backend/config/database.js`

```javascript
const mysql = require("mysql2");

const pool = mysql.createPool({
  host: process.env.DB_HOST || "localhost",
  user: process.env.DB_USER || "root",
  password: process.env.DB_PASSWORD || "",
  database: process.env.DB_NAME || "obraconnect",
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0,
});

const db = pool.promise();

module.exports = db;
```

Linha a linha:
- `mysql2`: driver do MySQL.
- `createPool`: cria pool de conexões para reuso e desempenho.
- `process.env.DB_HOST`: lê host do .env.
- `pool.promise()`: usa promessas await/async.
- `module.exports = db;`: exporta objeto de conexão para usar em rotas.

### 4.5. Crie `backend/config/upload.js`

```javascript
const multer = require("multer");
const path = require("path");

const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, "uploads/");
  },
  filename: (req, file, cb) => {
    const ext = path.extname(file.originalname); // .jpg
    const baseName = path.basename(file.originalname, ext);
    const uniqueName = `${baseName}-${Date.now()}${ext}`;
    cb(null, uniqueName);
  },
});

const fileFilter = (req, file, cb) => {
  const allowed = [".png", ".jpg", ".jpeg"];
  const ext = path.extname(file.originalname).toLowerCase();
  if (allowed.includes(ext)) {
    cb(null, true);
  } else {
    cb(new Error("Apenas imagens são permitidas (PNG, JPG, JPEG)."), false);
  }
};

const upload = multer({ storage, fileFilter, limits: { fileSize: 5 * 1024 * 1024 } });

module.exports = upload;
```

- `multer.diskStorage`: configura onde salvar e como nomear arquivo.
- `uploads/`: pasta local onde as imagens serão gravadas.
- `fileFilter`: filtra tipo de arquivo.
- `limits.fileSize`: limite 5MB.

### 4.6. Crie middleware `backend/middlewares/autenticacao.js`

```javascript
const jwt = require("jsonwebtoken");

function autenticacao(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader || !authHeader.startsWith("Bearer ")) {
    return res.status(401).json({ erro: "Token não fornecido ou inválido." });
  }

  const token = authHeader.split(" ")[1];

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (erro) {
    return res.status(403).json({ erro: "Token inválido ou expirado." });
  }
}

module.exports = autenticacao;
```

- `req.headers.authorization`: cabeçalho da requisição com token.
- `jwt.verify`: verifica assinatura e validade do token.
- `req.user = decoded`: adiciona dados do usuário à requisição para usar nas rotas.

### 4.7. Rotas passo a passo

Crie o seguinte em `backend/routes`: cada arquivo exporta `express.Router()`.

#### 4.7.1 `authRoutes.js`

```javascript
const express = require("express");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");
const db = require("../config/database");

const router = express.Router();

router.post("/register", async (req, res) => {
  const { nome, email, senha } = req.body;

  if (!nome || !email || !senha) {
    return res.status(400).json({ erro: "Todos os campos são obrigatórios." });
  }

  try {
    const [users] = await db.query("SELECT id FROM users WHERE email = ?", [email]);
    if (users.length > 0) {
      return res.status(409).json({ erro: "Usuário já existe." });
    }

    const senhaHash = await bcrypt.hash(senha, 10);

    const [result] = await db.query(
      "INSERT INTO users (nome, email, senha) VALUES (?, ?, ?)",
      [nome, email, senhaHash],
    );

    const token = jwt.sign({ id: result.insertId, nome, email }, process.env.JWT_SECRET, {
      expiresIn: "1h",
    });

    res.status(201).json({ mensagem: "Usuário criado", token });
  } catch (erro) {
    console.error(erro);
    res.status(500).json({ erro: "Erro interno." });
  }
});

router.post("/login", async (req, res) => {
  const { email, senha } = req.body;
  if (!email || !senha) {
    return res.status(400).json({ erro: "Email e senha são obrigatórios." });
  }

  try {
    const [users] = await db.query("SELECT * FROM users WHERE email = ?", [email]);
    if (users.length === 0) {
      return res.status(401).json({ erro: "Credenciais inválidas." });
    }

    const user = users[0];
    const senhaValida = await bcrypt.compare(senha, user.senha);
    if (!senhaValida) {
      return res.status(401).json({ erro: "Credenciais inválidas." });
    }

    const token = jwt.sign({ id: user.id, nome: user.nome, email: user.email }, process.env.JWT_SECRET, {
      expiresIn: "1h",
    });

    res.json({ mensagem: "Login bem-sucedido", token });
  } catch (erro) {
    console.error(erro);
    res.status(500).json({ erro: "Erro interno." });
  }
});

module.exports = router;
```

Explicação:
- `router.post('/register')`: cria usuário
- `db.query`: executa SQL
- `bcrypt.hash`: gera hash seguro de senha
- `jwt.sign`: cria token JWT

#### 4.7.2 `servicoRoutes.js`

```javascript
const express = require("express");
const db = require("../config/database");
const autenticacao = require("../middlewares/autenticacao");
const upload = require("../config/upload");

const router = express.Router();

router.get("/", async (req, res) => {
  try {
    const [servicos] = await db.query("SELECT s.*, u.nome AS usuario_nome FROM servicos s JOIN users u ON s.usuario_id = u.id ORDER BY s.criado_em DESC");
    res.json(servicos);
  } catch (erro) {
    console.error(erro);
    res.status(500).json({ erro: "Erro ao buscar serviços." });
  }
});

router.get("/:id", async (req, res) => {
  const { id } = req.params;
  try {
    const [servicos] = await db.query("SELECT s.*, u.nome AS usuario_nome FROM servicos s JOIN users u ON s.usuario_id = u.id WHERE s.id = ?", [id]);
    if (servicos.length === 0) {
      return res.status(404).json({ erro: "Serviço não encontrado." });
    }
    res.json(servicos[0]);
  } catch (erro) {
    console.error(erro);
    res.status(500).json({ erro: "Erro ao buscar serviço." });
  }
});

router.post("/", autenticacao, upload.single("imagem"), async (req, res) => {
  const { titulo, descricao, preco } = req.body;
  const usuarioId = req.user.id;
  const imagem = req.file ? req.file.filename : null;

  if (!titulo || !descricao || !preco) {
    return res.status(400).json({ erro: "Título, descrição e preço são obrigatórios." });
  }

  try {
    const [result] = await db.query(
      "INSERT INTO servicos (titulo, descricao, preco, imagem, usuario_id) VALUES (?, ?, ?, ?, ?)",
      [titulo, descricao, preco, imagem, usuarioId],
    );

    res.status(201).json({ mensagem: "Serviço criado", id: result.insertId });
  } catch (erro) {
    console.error(erro);
    res.status(500).json({ erro: "Erro ao criar serviço." });
  }
});

router.put("/:id", autenticacao, async (req, res) => {
  const { id } = req.params;
  const { titulo, descricao, preco } = req.body;
  const usuarioId = req.user.id;

  try {
    const [servicos] = await db.query("SELECT * FROM servicos WHERE id = ?", [id]);
    if (servicos.length === 0) {
      return res.status(404).json({ erro: "Serviço não encontrado." });
    }

    const servico = servicos[0];
    if (servico.usuario_id !== usuarioId) {
      return res.status(403).json({ erro: "Acesso negado." });
    }

    await db.query(
      "UPDATE servicos SET titulo = ?, descricao = ?, preco = ? WHERE id = ?",
      [titulo || servico.titulo, descricao || servico.descricao, preco || servico.preco, id],
    );

    res.json({ mensagem: "Serviço atualizado" });
  } catch (erro) {
    console.error(erro);
    res.status(500).json({ erro: "Erro ao atualizar serviço." });
  }
});

router.delete("/:id", autenticacao, async (req, res) => {
  const { id } = req.params;
  const usuarioId = req.user.id;

  try {
    const [servicos] = await db.query("SELECT * FROM servicos WHERE id = ?", [id]);
    if (servicos.length === 0) {
      return res.status(404).json({ erro: "Serviço não encontrado." });
    }

    const servico = servicos[0];
    if (servico.usuario_id !== usuarioId) {
      return res.status(403).json({ erro: "Acesso negado." });
    }

    await db.query("DELETE FROM servicos WHERE id = ?", [id]);
    res.json({ mensagem: "Serviço removido" });
  } catch (erro) {
    console.error(erro);
    res.status(500).json({ erro: "Erro ao excluir serviço." });
  }
});

module.exports = router;
```

#### 4.7.3 `avaliacaoRoutes.js`

```javascript
const express = require("express");
const db = require("../config/database");
const autenticacao = require("../middlewares/autenticacao");

const router = express.Router();

router.post("/", autenticacao, async (req, res) => {
  const { servico_id, nota, comentario } = req.body;
  const usuarioId = req.user.id;

  if (!servico_id || !nota) {
    return res.status(400).json({ erro: "Servico_id e nota são obrigatórios." });
  }

  try {
    await db.query(
      "INSERT INTO avaliacoes (servico_id, usuario_id, nota, comentario) VALUES (?, ?, ?, ?)",
      [servico_id, usuarioId, nota, comentario],
    );
    res.status(201).json({ mensagem: "Avaliação criada" });
  } catch (erro) {
    console.error(erro);
    res.status(500).json({ erro: "Erro ao cadastrar avaliação." });
  }
});

router.get("/servico/:servicoId", async (req, res) => {
  const { servicoId } = req.params;

  try {
    const [avaliacoes] = await db.query(
      "SELECT a.*, u.nome AS usuario_nome FROM avaliacoes a JOIN users u ON a.usuario_id = u.id WHERE a.servico_id = ? ORDER BY a.criado_em DESC",
      [servicoId],
    );
    res.json(avaliacoes);
  } catch (erro) {
    console.error(erro);
    res.status(500).json({ erro: "Erro ao buscar avaliações." });
  }
});

module.exports = router;
```

#### 4.7.4 `testesRoutes.js`

```javascript
const express = require("express");
const router = express.Router();

router.get("/ping", (req, res) => {
  res.json({ mensagem: "pong" });
});

module.exports = router;
```

---

## 5. Frontend: arquivos e funcionalidades

### 5.1. `frontend/index.html`

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8" />
  <meta http-equiv="X-UA-Compatible" content="IE=edge" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>obraConnect</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <header>
    <h1>obraConnect</h1>
    <nav>
      <a href="/frontend/index.html">Home</a>
      <a href="/frontend/pages/login.html">Login</a>
      <a href="/frontend/pages/registro.html">Registro</a>
      <a href="/frontend/pages/meu-perfil.html">Meu perfil</a>
    </nav>
  </header>
  <main>
    <section id="lista-servicos"></section>
  </main>
  <script src="js/api.js"></script>
  <script src="js/servicos.js"></script>
</body>
</html>
```

- `meta viewport`: torna responsivo em mobile.
- `link rel` para `style.css`.
- `script` importa funções JS.

### 5.2. `frontend/style.css`

(explicação de estilos básicos). Exemplo:

```css
body { font-family: Arial, sans-serif; margin: 0; padding: 0; background: #f7f7f7; }
header { background: #004aad; color: white; padding: 15px; }
header nav a { color: white; margin-right: 15px; text-decoration: none; }
main { max-width: 1100px; margin: 20px auto; padding: 0 20px; }
.servico-card { background: white; border-radius: 8px; padding: 15px; margin-bottom: 10px; box-shadow: 0 1px 4px rgba(0,0,0,0.12); }
```

### 5.3. `frontend/js/api.js`

```javascript
const API_BASE_URL = "http://localhost:3001/api";

function buildHeaders(token, hasBody = true) {
  const headers = {};
  if (hasBody) {
    headers["Content-Type"] = "application/json";
  }
  if (token) {
    headers["Authorization"] = `Bearer ${token}`;
  }
  return headers;
}

async function request(path, method = "GET", body = null, token = null) {
  const options = {
    method,
    headers: buildHeaders(token, body !== null),
  };

  if (body !== null) {
    options.body = JSON.stringify(body);
  }

  const response = await fetch(`${API_BASE_URL}${path}`, options);
  const data = await response.json();

  if (!response.ok) {
    throw new Error(data.erro || data.message || "Erro desconhecido");
  }

  return data;
}

async function login(email, senha) {
  return request("/auth/login", "POST", { email, senha });
}

async function register(nome, email, senha) {
  return request("/auth/register", "POST", { nome, email, senha });
}

async function listarServicos() {
  return request("/servicos", "GET");
}

async function buscarServico(id) {
  return request(`/servicos/${id}`, "GET");
}

async function criarServico(formData) {
  const token = localStorage.getItem("token");
  const response = await fetch(`${API_BASE_URL}/servicos`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${token}`,
    },
    body: formData,
  });
  if (!response.ok) {
    const erro = await response.json();
    throw new Error(erro.erro || "Erro ao criar serviço");
  }
  return response.json();
}

async function avaliarServico(servico_id, nota, comentario) {
  const token = localStorage.getItem("token");
  return request("/avaliacoes", "POST", { servico_id, nota, comentario }, token);
}

async function listarAvaliacoes(servico_id) {
  return request(`/avaliacoes/servico/${servico_id}`, "GET");
}
```

Explicações:
- `API_BASE_URL`: base da API backend.
- `buildHeaders`: monta cabeçalhos, inserindo `Content-Type` e `Authorization`.
- `request`: faz fetch e trata erros.
- `criarServico`: usa `FormData` e não define `Content-Type`; o navegador define multipart/form-data automáticamente.

### 5.4. `frontend/js/auth.js`

```javascript
const formLogin = document.getElementById("form-login");
if (formLogin) {
  formLogin.addEventListener("submit", async (event) => {
    event.preventDefault();
    const email = document.getElementById("email").value;
    const senha = document.getElementById("senha").value;

    try {
      const resultado = await login(email, senha);
      localStorage.setItem("token", resultado.token);
      window.location.href = "/frontend/index.html";
    } catch (erro) {
      alert("Falha no login: " + erro.message);
    }
  });
}

const formRegistro = document.getElementById("form-registro");
if (formRegistro) {
  formRegistro.addEventListener("submit", async (event) => {
    event.preventDefault();
    const nome = document.getElementById("nome").value;
    const email = document.getElementById("email").value;
    const senha = document.getElementById("senha").value;

    try {
      const resultado = await register(nome, email, senha);
      localStorage.setItem("token", resultado.token);
      window.location.href = "/frontend/index.html";
    } catch (erro) {
      alert("Falha no registro: " + erro.message);
    }
  });
}

function logout() {
  localStorage.removeItem("token");
  window.location.href = "/frontend/pages/login.html";
}
```

### 5.5. `frontend/js/servicos.js`

```javascript
async function carregarServicos() {
  try {
    const servicos = await listarServicos();
    const lista = document.getElementById("lista-servicos");
    lista.innerHTML = "";

    servicos.forEach((servico) => {
      const card = document.createElement("div");
      card.className = "servico-card";
      card.innerHTML = `
        <h2>${servico.titulo}</h2>
        <p>${servico.descricao}</p>
        <p><strong>R$ ${servico.preco}</strong></p>
        <p>Por: ${servico.usuario_nome}</p>
        <a href="/frontend/pages/detalhes-servico.html?id=${servico.id}">Ver detalhes</a>
      `;
      lista.appendChild(card);
    });
  } catch (erro) {
    console.error("Erro ao listar serviços", erro);
    document.getElementById("lista-servicos").innerText = "Erro ao carregar serviços.";
  }
}

if (document.getElementById("lista-servicos")) {
  carregarServicos();
}
```

- `carregarServicos` chama backend e popula DOM.

### 5.6. Página de cadastro de serviço `frontend/pages/cadastrar-servico.html`

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Cadastrar serviço</title>
  <link rel="stylesheet" href="../style.css" />
</head>
<body>
  <h1>Cadastrar serviço</h1>
  <form id="form-cadastrar-servico">
    <input type="text" id="titulo" placeholder="Título" required />
    <textarea id="descricao" placeholder="Descrição" required></textarea>
    <input type="number" step="0.01" id="preco" placeholder="Preço" required />
    <input type="file" id="imagem" accept="image/*" />
    <button type="submit">Enviar</button>
  </form>
  <script src="../js/api.js"></script>
  <script>
    const form = document.getElementById("form-cadastrar-servico");
    form.addEventListener("submit", async (event) => {
      event.preventDefault();

      const titulo = document.getElementById("titulo").value;
      const descricao = document.getElementById("descricao").value;
      const preco = document.getElementById("preco").value;
      const imagemInput = document.getElementById("imagem");

      const formData = new FormData();
      formData.append("titulo", titulo);
      formData.append("descricao", descricao);
      formData.append("preco", preco);
      if (imagemInput.files.length > 0) {
        formData.append("imagem", imagemInput.files[0]);
      }

      try {
        await criarServico(formData);
        alert("Serviço cadastrado com sucesso!");
        window.location.href = "/frontend/index.html";
      } catch (erro) {
        alert("Erro ao cadastrar serviço: " + erro.message);
      }
    });
  </script>
</body>
</html>
```

- `FormData` é usado para enviar imagem; diferente de JSON.

### 5.7. Página detalhes do serviço `frontend/pages/detalhes-servico.html`

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Detalhes do serviço</title>
  <link rel="stylesheet" href="../style.css" />
</head>
<body>
  <main>
    <div id="detalhes"></div>
    <div id="avaliacoes"></div>
    <form id="form-avaliar">
      <input type="number" id="nota" min="1" max="5" required placeholder="Nota 1 a 5" />
      <textarea id="comentario" placeholder="Comentário (opcional)"></textarea>
      <button type="submit">Enviar avaliação</button>
    </form>
  </main>

  <script src="../js/api.js"></script>
  <script>
    function getQueryParam(name) {
      const urlParams = new URLSearchParams(window.location.search);
      return urlParams.get(name);
    }

    async function carregarDetalhes() {
      const id = getQueryParam("id");
      if (!id) {
        document.getElementById("detalhes").innerText = "ID do serviço não informado.";
        return;
      }

      try {
        const servico = await buscarServico(id);
        document.getElementById("detalhes").innerHTML = `
          <h2>${servico.titulo}</h2>
          <p>${servico.descricao}</p>
          <p><strong>R$ ${servico.preco}</strong></p>
          <p>Autor: ${servico.usuario_nome}</p>
          ${servico.imagem ? `<img src="http://localhost:3001/uploads/${servico.imagem}" alt="Imagem do serviço" style="max-width: 350px;" />` : ""}
        `;

        const avaliacoes = await listarAvaliacoes(id);
        const avaliacoesDiv = document.getElementById("avaliacoes");
        avaliacoesDiv.innerHTML = "<h3>Avaliações</h3>";
        if (avaliacoes.length === 0) {
          avaliacoesDiv.innerHTML += "<p>Nenhuma avaliação ainda.</p>";
        } else {
          avaliacoes.forEach((a) => {
            avaliacoesDiv.innerHTML += `<div><strong>${a.usuario_nome}</strong> (${a.nota}/5): ${a.comentario || "Sem comentário"}</div>`;
          });
        }
      } catch (erro) {
        console.error(erro);
        document.getElementById("detalhes").innerText = "Erro ao buscar serviço.";
      }
    }

    document.getElementById("form-avaliar").addEventListener("submit", async (event) => {
      event.preventDefault();
      const servico_id = getQueryParam("id");
      const nota = Number(document.getElementById("nota").value);
      const comentario = document.getElementById("comentario").value;

      try {
        await avaliarServico(servico_id, nota, comentario);
        alert("Avaliação enviada!");
        carregarDetalhes();
      } catch (erro) {
        alert("Erro ao enviar avaliação: " + erro.message);
      }
    });

    carregarDetalhes();
  </script>
</body>
</html>
```

---

## 6. Testar e executar

1. Inicie backend:
```bash
cd backend
npx nodemon index.js
```
2. Abra banco de dados e verifique dados.
3. Rode frontend (pode abrir `frontend/index.html` diretamente ou usar servidor local):
```bash
cd frontend
npx serve .
```
4. Acesse `http://localhost:3000` (ou se abriu direto em arquivo, use `file://`).
5. Cadastre usuário, login, crie serviço, veja detalhes, avalie.

---

## 7. Explicação de cada conceito de forma didática

- **Variáveis**: `const` para valores fixos, `let` para mutáveis.
- **Funções**: `function nome()` ou `const nome = () => {}`.
- **Async/Await**: permite escrever código assíncrono como se fosse síncrono.
- **Objeto**: `{ chave: valor }`.
- **Array**: lista ordenada de valores.
- **JSON**: formato text-based para enviar dados (`{}` e `[]`).
- **HTTP**: protocolo de comunicação entre frontend e backend.
- **métodos HTTP**: `GET` (ler), `POST` (criar), `PUT` (atualizar), `DELETE` (apagar).
- **Status code**: 200 ok, 201 criado, 400 bad request, 401 não autorizado, 403 proibido, 404 não encontrado, 500 erro servidor.
- **JWT**: token assinado que carrega dados do usuário para autenticar.
- **CORS**: política de segurança do navegador que bloqueia requisições cross-origin sem permissão.
- **Middleware**: função que roda antes da rota para executar lógica geral (ex: autenticação).

---

## 8. Checklist para replicar

- [ ] Criar banco e tabelas
- [ ] Configurar `.env`
- [ ] Criar backend `index.js`, config, middlewares, rotas
- [ ] Instalar dependências Node
- [ ] Iniciar backend
- [ ] Criar frontend HTML/CSS/JS
- [ ] Testar cadastro e login
- [ ] Testar CRUD de serviço
- [ ] Testar avaliações


---

## 9. Como ajustar quando algo quebrar

- Logs do backend: revise console do terminal.
- Erro 404 em rota: verificar URL de `fetch` e prefixo `/api/...`.
- Erro de CORS: garante `app.use(cors({ origin: "*" }));`.
- Erro ao conectar MySQL: revisa `.env` e nomes de DB/tabela.
- Token JWT expira: reinicie login.


> Pronto: agora você possui um passo a passo detalhado, em markdown, pronto para seguir e entender cada substância do projeto.