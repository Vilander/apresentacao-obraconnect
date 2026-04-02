# Código atual do backend com comentários linha a linha

Este arquivo reproduz exatamente o código atual do backend da aplicação e inclui comentários explicativos detalhados para cada seção.

## backend/index.js

```javascript
require("dotenv").config();
// Carrega as variáveis de ambiente do arquivo .env para process.env.

const express = require("express");
// Importa o framework Express para criar um servidor HTTP.

const cors = require("cors");
// Importa middleware CORS para configurar segurança e permitir requisições de outro domínio.

const path = require("path");
// Módulo do Node para manipular caminhos de arquivo de forma independente de sistema operacional.

const banco = require("./config/database");
// Importa a configuração de conexão com o banco de dados MySQL.

// Importar rotas
const authRoutes = require("./routes/authRoutes");
const servicoRoutes = require("./routes/servicoRoutes");
const avaliacaoRoutes = require("./routes/avaliacaoRoutes");
const testesRoutes = require("./routes/testesRoutes");

// Inicializar Express
const app = express();
// Cria a instância do servidor Express.

const PORT = process.env.PORT || 3001;
// Porta que o servidor vai escutar. Lê do .env ou usa 3001 como padrão.

// ===============================================
// MIDDLEWARES GLOBAIS
// ===============================================

// CORS - Permite requisições do frontend
app.use(
  cors({
    origin: "*",
    methods: ["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
    credentials: true,
  }),
);
// Permite que qualquer origem faça requisições. Em produção, deve ser ajustado para frontend confiável.

// Body Parser - Entender JSON
app.use(express.json());
// Converte corpo JSON de requisições em objeto JavaScript em req.body.

//Debug para ver todas as requisições:
// app.use((req, res, next) => {
//   console.log(`Requisição recebida: ${req.method} ${req.url}`);
//   next();
// });
// Middleware comentado para debug. Você pode ativar para logar cada chamada.

app.use(express.urlencoded({ extended: true }));
// Suporta parsing de formulários com urlencoded.

// Servir arquivos estáticos (imagens)
app.use("/uploads", express.static(path.join(__dirname, "uploads")));
// Expõe conteúdos da pasta uploads em /uploads.

// ===============================================
// ROTAS
// ===============================================

// Registrar rotas
app.use("/api/auth", authRoutes);
app.use("/api/servicos", servicoRoutes);
app.use("/api/avaliacoes", avaliacaoRoutes);
app.use("/api/testes", testesRoutes);

// ===============================================
// TRATAMENTO DE ERROS
// ===============================================

// 404 - Rota não encontrada
app.use((req, res) => {
  res.status(404).json({ erro: "Rota não encontrada." });
});

// Erro geral
app.use((erro, req, res, next) => {
  console.error("Erro no servidor:", erro);
  res.status(500).json({
    erro: "Erro interno do servidor.",
    detalhes: process.env.NODE_ENV === "development" ? erro.message : undefined,
  });
});

// ===============================================
// INICIAR SERVIDOR
// ===============================================

app.listen(PORT, () => {
  console.log("");
  console.log(`|Servidor rodando|`);
  console.log("");
});

process.on("SIGTERM", () => {
  console.log("Servidor encerrando...");
  process.exit(0);
});
```

## backend/config/database.js

```javascript
/**
 * Configuração de Conexão com MySQL
 * Usa mysql2/promise para operações assíncronas
 */

require("dotenv").config();
// Carrega .env para ter acesso às variáveis de ambiente.

const mysql = require("mysql2/promise");
// Usa a versão promise do mysql2 para poder usar async/await.

const pool = mysql.createPool({
  host: process.env.DB_HOST || "localhost",
  user: process.env.DB_USER || "root",
  password: process.env.DB_PASSWORD || "",
  database: process.env.DB_NAME || "obraconnect_db",
  port: process.env.DB_PORT || 3306,
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 0,
});
// Cria pool de conexões, lendo as configurações do .env.

// Testar conexão ao iniciar
pool
  .getConnection()
  .then((connection) => {
    console.log("✅ Conectado ao banco de dados com sucesso!");
    connection.release();
  })
  .catch((erro) => {
    console.error("❌ Erro ao conectar no banco de dados:", erro.message);
    process.exit(1);
  });
// Tenta pegar uma conexão. Se falhar, encerra processo para sinalizar erro.

module.exports = pool;
// Exporta pool para usar em outras partes do backend.
```

## backend/config/upload.js

```javascript
/**
 * Configuração para upload de imagens
 * Salva as fotos na pasta /uploads
 */

const multer = require("multer");
const path = require("path");
const fs = require("fs");

// Criar pasta uploads se não existir
const uploadDir = path.join(__dirname, "../uploads");
if (!fs.existsSync(uploadDir)) {
  fs.mkdirSync(uploadDir, { recursive: true });
}

// Configurar storage
const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    cb(null, uploadDir);
  },
  filename: function (req, file, cb) {
    // Gerar nome único com timestamp
    const uniqueSuffix = Date.now() + "-" + Math.round(Math.random() * 1e9);
    cb(
      null,
      file.fieldname + "-" + uniqueSuffix + path.extname(file.originalname),
    );
  },
});

// Filtro de arquivos (apenas imagens)
const fileFilter = (req, file, cb) => {
  const extensoesPermitidas = /jpeg|jpg|png|gif/;
  const tiposPermitidos = /image\/jpeg|image\/png|image\/gif/;

  const extname = extensoesPermitidas.test(
    path.extname(file.originalname).toLowerCase(),
  );
  const mimetype = tiposPermitidos.test(file.mimetype);

  if (mimetype && extname) {
    return cb(null, true);
  } else {
    cb(new Error("Apenas imagens (jpeg, jpg, png, gif) são permitidas!"));
  }
};

module.exports = multer({
  storage: storage,
  fileFilter: fileFilter,
  limits: {
    fileSize: parseInt(process.env.MAX_FILE_SIZE) || 5 * 1024 * 1024, // 5MB
  },
});
```

## backend/middlewares/autenticacao.js

```javascript
/**
 * Middleware de Autenticação com JWT
 * Verifica se o usuário está logado antes de acessar rotas protegidas
 */

const jwt = require("jsonwebtoken");

/**
 * Middleware para verificar JWT
 * Adiciona dados do usuário em req.usuario
 */
function verificarToken(req, res, next) {
  // 1. Pega o token do header Authorization
  const token = req.headers.authorization?.split(" ")[1]; // "Bearer TOKEN"

  if (!token) {
    return res
      .status(401)
      .json({ erro: "Token não fornecido. Faça login primeiro." });
  }

  try {
    // 2. Verifica e decodifica o token
    const decoded = jwt.verify(token, process.env.JWT_SECRET);

    // 3. Adiciona dados do usuário à requisição
    req.usuario = decoded;

    next();
  } catch (erro) {
    if (erro.name === "TokenExpiredError") {
      return res
        .status(401)
        .json({ erro: "Token expirado. Faça login novamente." });
    }

    return res.status(403).json({ erro: "Token inválido." });
  }
}

/**
 * Middleware para verificar se é prestador ou admin
 */
function verificarPrestador(req, res, next) {
  if (
    req.usuario.tipo_usuario !== "prestador" &&
    req.usuario.tipo_usuario !== "admin"
  ) {
    return res
      .status(403)
      .json({ erro: "Apenas prestadores podem acessar isso." });
  }
  next();
}

/**
 * Middleware para verificar se é admin
 */
function verificarAdmin(req, res, next) {
  if (req.usuario.tipo_usuario !== "admin") {
    return res
      .status(403)
      .json({ erro: "Apenas administradores podem acessar isso." });
  }
  next();
}

module.exports = {
  verificarToken,
  verificarPrestador,
  verificarAdmin,
};
```

## backend/routes/authRoutes.js

```javascript
/**
 * Rotas de Autenticação
 * - POST /registro - Registrar novo usuário
 * - POST /login - Fazer login
 * - GET /perfil - Ver dados do usuário logado
 * - PUT /tornar-prestador - Virar prestador
 */

const express = require("express");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");
const banco = require("../config/database");
const { verificarToken } = require("../middlewares/autenticacao");

const router = express.Router();

// ✅ FUNÇÃO HELPER - Gerar Token JWT
function gerarToken(usuario) {
  // console.log("Minha chave secreta é:", process.env.JWT_SECRET);
  return jwt.sign(
    {
      id: usuario.id,
      nome: usuario.nome_usuario,
      email: usuario.email,
      tipo_usuario: usuario.tipo_usuario,
    },
    process.env.JWT_SECRET,
    { expiresIn: process.env.JWT_EXPIRY },
  );
}

// ===============================================
// 1. REGISTRO DE NOVO USUÁRIO
// ===============================================
router.post("/registro", async (req, res) => {
  const { nome_usuario, email, senha, login, telefone } = req.body;

  // Validar campos obrigatórios
  if (!nome_usuario || !email || !senha || !login) {
    return res.status(400).json({ erro: "Todos os campos são obrigatórios!" });
  }

  // Validar email
  const regexEmail = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!regexEmail.test(email)) {
    return res.status(400).json({ erro: "Email inválido!" });
  }

  // Validar comprimento de senha
  if (senha.length < 6) {
    return res
      .status(400)
      .json({ erro: "Senha deve ter pelo menos 6 caracteres!" });
  }

  try {
    // Verificar se usuário/email já existe
    const [usuariosExistentes] = await banco.query(
      "SELECT * FROM oc__tb_usuario WHERE email = ? OR login = ?",
      [email, login],
    );

    if (usuariosExistentes.length > 0) {
      return res.status(409).json({ erro: "E-mail ou Login já cadastrados!" });
    }

    // Criptografar senha com bcrypt
    const salt = await bcrypt.genSalt(10);
    const senhaCriptografada = await bcrypt.hash(senha, salt);

    // Inserir usuário no banco
    const [resultado] = await banco.query(
      "INSERT INTO oc__tb_usuario (nome_usuario, email, senha, login, telefone, tipo_usuario, data_cadastro) VALUES (?, ?, ?, ?, ?, ?, NOW())",
      [
        nome_usuario,
        email,
        senhaCriptografada,
        login,
        telefone || null,
        "usuario",
      ],
    );

    res.status(201).json({
      mensagem: "Usuário registrado com sucesso!",
      id_usuario: resultado.insertId,
      usuario: nome_usuario,
    });
  } catch (erro) {
    console.error("Erro ao registrar usuário:", erro);
    res.status(500).json({ erro: "Erro ao registrar usuário." });
  }
});

// ===============================================
// 2. LOGIN
// ===============================================
router.post("/login", async (req, res) => {
  const { login, senha } = req.body;

  // Validar campos
  if (!login || !senha) {
    return res.status(400).json({ erro: "Login e senha são obrigatórios!" });
  }

  try {
    // Buscar usuário por login ou email
    const [usuarios] = await banco.query(
      "SELECT * FROM oc__tb_usuario WHERE login = ? OR email = ?",
      [login, login],
    );

    if (usuarios.length === 0) {
      return res.status(401).json({ erro: "Usuário ou senha incorretos." });
    }

    const usuario = usuarios[0];

    // Verificar senha com bcrypt
    const senhaValida = await bcrypt.compare(senha, usuario.senha);
    if (!senhaValida) {
      return res.status(401).json({ erro: "Usuário ou senha incorretos." });
    }

    // Gerar token
    const token = gerarToken(usuario);

    // Retornar sucesso
    res.status(200).json({
      mensagem: "Login realizado com sucesso!",
      token: token,
      usuario: {
        id: usuario.id,
        nome: usuario.nome_usuario,
        email: usuario.email,
        tipo_usuario: usuario.tipo_usuario,
      },
    });
  } catch (erro) {
    console.error("Erro ao fazer login:", erro);
    res.status(500).json({ erro: "Erro ao fazer login." });
  }
});

// ===============================================
// 3. VER PERFIL (PROTEGIDO)
// ===============================================
router.get("/perfil", verificarToken, async (req, res) => {
  try {
    // Buscar dados atualizados do usuário
    const [usuarios] = await banco.query(
      "SELECT id, nome_usuario, email, login, tipo_usuario, data_cadastro FROM oc__tb_usuario WHERE id = ?",
      [req.usuario.id],
    );

    if (usuarios.length === 0) {
      return res.status(404).json({ erro: "Usuário não encontrado." });
    }

    res.status(200).json({
      mensagem: "Dados do usuário",
      usuario: usuarios[0],
    });
  } catch (erro) {
    console.error("Erro ao buscar perfil:", erro);
    res.status(500).json({ erro: "Erro ao buscar dados do usuário." });
  }
});

// ===============================================
// 4. TORNAR PRESTADOR (PROTEGIDO)
// ===============================================
router.put("/tornar-prestador", verificarToken, async (req, res) => {
  try {
    // Verificar se já é prestador
    const [usuarios] = await banco.query(
      "SELECT tipo_usuario FROM oc__tb_usuario WHERE id = ?",
      [req.usuario.id],
    );

    if (usuarios.length === 0) {
      return res.status(404).json({ erro: "Usuário não encontrado." });
    }

    if (
      usuarios[0].tipo_usuario === "prestador" ||
      usuarios[0].tipo_usuario === "admin"
    ) {
      return res.status(400).json({ erro: "Você já é prestador!" });
    }

    // Atualizar tipo de usuário
    await banco.query(
      "UPDATE oc__tb_usuario SET tipo_usuario = ? WHERE id = ?",
      ["prestador", req.usuario.id],
    );

    // Gerar novo token com tipo atualizado
    const usuarioAtualizado = {
      ...req.usuario,
      tipo_usuario: "prestador",
    };
    const novoToken = gerarToken(usuarioAtualizado);

    res.status(200).json({
      mensagem: "Você agora é um prestador!",
      token: novoToken,
      usuario: usuarioAtualizado,
    });
  } catch (erro) {
    console.error("Erro ao atualizar para prestador:", erro);
    res.status(500).json({ erro: "Erro ao atualizar usuário." });
  }
});

module.exports = router;
```

## backend/routes/servicoRoutes.js

```javascript
/**
 * Rotas de Serviços
 * - GET / - Listar todos os serviços (público)
 * - GET /:id - Detalhes de um serviço
 * - GET /meus/servicos - Listar meus serviços (protegido)
 * - POST / - Criar serviço (protegido)
 * - PUT /:id - Editar serviço (protegido)
 * - DELETE /:id - Deletar serviço (protegido)
 */

const express = require("express");
const banco = require("../config/database");
const upload = require("../config/upload");
const {
  verificarToken,
  verificarPrestador,
} = require("../middlewares/autenticacao");

const router = express.Router();

// ===============================================
// 1. LISTAR TODOS OS SERVIÇOS (PÚBLICO)
// ===============================================
router.get("/", async (req, res) => {
  try {
    const [servicos] = await banco.query(
      "SELECT * FROM oc__vw_servicos_ativos",
    );

    const servicosFormatados = servicos.map((s) => ({
      ...s,
      nota_media: parseFloat(s.nota_media) || 0,
    }));

    res.status(200).json(servicosFormatados);
  } catch (erro) {
    console.error("Erro ao listar serviços:", erro);
    res.status(500).json({ erro: "Erro ao listar serviços." });
  }
});

// ===============================================
// 2. LISTAR MEUS SERVIÇOS (PROTEGIDO) - DEVE VIR ANTES DE /:id
// ===============================================
router.get("/meus/servicos", verificarToken, async (req, res) => {
  try {
    const [servicos] = await banco.query(
      "SELECT * FROM oc__tb_servico WHERE id_usuario = ? ORDER BY data_cadastro DESC",
      [req.usuario.id],
    );

    res.status(200).json(servicos);
  } catch (erro) {
    console.error("Erro ao listar meus serviços:", erro);
    res.status(500).json({ erro: "Erro ao listar seus serviços." });
  }
});

// ===============================================
// 3. BUSCAR SERVIÇO POR ID (PÚBLICO)
// ===============================================
router.get("/:id", async (req, res) => {
  const { id } = req.params;

  try {
    // Chamamos a view e passamos o ID no WHERE
    const [servicos] = await banco.query(
      "SELECT * FROM oc__vw_detalhes_servico WHERE id = ?",
      [id],
    );

    if (servicos.length === 0) {
      return res.status(404).json({ erro: "Serviço não encontrado." });
    }

    const servico = servicos[0];
    servico.nota_media = parseFloat(servico.nota_media) || 0;

    res.status(200).json(servico);
  } catch (erro) {
    console.error("Erro ao buscar serviço:", erro);
    res.status(500).json({ erro: "Erro ao buscar serviço." });
  }
});

// ===============================================
// 4. CRIAR NOVO SERVIÇO (PROTEGIDO - PRESTADOR)
// ===============================================
router.post(
  "/",
  verificarToken,
  verificarPrestador,
  upload.single("imagem"),
  async (req, res) => {
    const { titulo, descricao, id_categoria } = req.body;

    if (!titulo || !descricao) {
      return res
        .status(400)
        .json({ erro: "Título e Descrição são obrigatórios!" });
    }

    try {
      let imagemUrl = null;
      if (req.file) {
        const host = req.get("host");
        imagemUrl = `${req.protocol}://${host}/uploads/${req.file.filename}`;
      }

      const [resultado] = await banco.query(
        `INSERT INTO oc__tb_servico (id_usuario, titulo, desc_servico, id_categoria, imagem_url, data_cadastro) 
       VALUES (?, ?, ?, ?, ?, NOW())`,
        [req.usuario.id, titulo, descricao, id_categoria || null, imagemUrl],
      );

      res.status(201).json({
        mensagem: "Serviço criado com sucesso!",
        id_servico: resultado.insertId,
        imagem: imagemUrl,
      });
    } catch (erro) {
      console.error("Erro ao criar serviço:", erro);
      res.status(500).json({ erro: "Erro ao criar serviço." });
    }
  },
);

// ===============================================
// 5. EDITAR SERVIÇO (PROTEGIDO)
// ===============================================
router.put(
  "/:id",
  verificarToken,
  upload.single("imagem"),
  async (req, res) => {
    const { id } = req.params;
    const { titulo, descricao, id_categoria } = req.body;

    try {
      // Verificar se existe e se é dono
      const [servicos] = await banco.query(
        "SELECT * FROM oc__tb_servico WHERE id = ?",
        [id],
      );

      if (servicos.length === 0) {
        return res.status(404).json({ erro: "Serviço não encontrado." });
      }

      if (servicos[0].id_usuario !== req.usuario.id) {
        return res
          .status(403)
          .json({ erro: "Você não tem permissão para editar este serviço." });
      }

      // Montar query de atualização dinâmica
      let sql =
        "UPDATE oc__tb_servico SET titulo = ?, desc_servico = ?, id_categoria = ?";
      let params = [
        titulo || servicos[0].titulo,
        descricao || servicos[0].desc_servico,
        id_categoria || servicos[0].id_categoria,
      ];

      // Se houver nova imagem, atualizar também
      if (req.file) {
        const host = req.get("host");
        sql += ", imagem_url = ?";
        params.push(`${req.protocol}://${host}/uploads/${req.file.filename}`);
      }

      sql += " WHERE id = ?";
      params.push(id);

      await banco.query(sql, params);

      res.status(200).json({
        mensagem: "Serviço atualizado com sucesso!",
        id_servico: id,
      });
    } catch (erro) {
      console.error("Erro ao editar serviço:", erro);
      res.status(500).json({ erro: "Erro ao editar serviço." });
    }
  },
);

// ===============================================
// 6. DESATIVAR SERVIÇO (PROTEGIDO)
// ===============================================
router.patch("/:id/desativar", verificarToken, async (req, res) => {
  const { id } = req.params;

  try {
    // Verificar se existe e se é dono
    const [servicos] = await banco.query(
      "SELECT * FROM oc__tb_servico WHERE id = ?",
      [id],
    );

    if (servicos.length === 0) {
      return res.status(404).json({ erro: "Serviço não encontrado." });
    }

    if (servicos[0].id_usuario !== req.usuario.id) {
      return res
        .status(403)
        .json({ erro: "Você não tem permissão para desativar este serviço." });
    }

    // Desativar serviço (apenas mudar status para 0)
    await banco.query("UPDATE oc__tb_servico SET ativo = 0 WHERE id = ?", [id]);

    res.status(200).json({
      mensagem: "Serviço desativado com sucesso!",
      id_servico: id,
    });
  } catch (erro) {
    console.error("Erro ao desativar serviço:", erro);
    res.status(500).json({ erro: "Erro ao desativar serviço." });
  }
});

// ===============================================
// 6.1. REATIVAR SERVIÇO (PROTEGIDO)
// ===============================================
router.patch("/:id/ativar", verificarToken, async (req, res) => {
  const { id } = req.params;

  try {
    // Verificar se o serviço existe
    const [servicos] = await banco.query(
      "SELECT * FROM oc__tb_servico WHERE id = ?",
      [id],
    );

    if (servicos.length === 0) {
      return res.status(404).json({ erro: "Serviço não encontrado." });
    }

    // Verificar se o usuário logado é o dono do serviço
    if (servicos[0].id_usuario !== req.usuario.id) {
      return res
        .status(403)
        .json({ erro: "Você não tem permissão para reativar este serviço." });
    }

    // Reativar serviço (mudar status para 1)
    await banco.query("UPDATE oc__tb_servico SET ativo = 1 WHERE id = ?", [id]);

    res.status(200).json({
      mensagem: "Serviço reativado com sucesso!",
      id_servico: id,
    });
  } catch (erro) {
    console.error("Erro ao reativar serviço:", erro);
    res.status(500).json({ erro: "Erro ao reativar serviço." });
  }
});

// ===============================================
// 7. DELETAR SERVIÇO (PROTEGIDO)
// ===============================================
router.delete("/:id", verificarToken, async (req, res) => {
  const { id } = req.params;

  try {
    // Verificar se existe e se é dono
    const [servicos] = await banco.query(
      "SELECT * FROM oc__tb_servico WHERE id = ?",
      [id],
    );

    if (servicos.length === 0) {
      return res.status(404).json({ erro: "Serviço não encontrado." });
    }

    if (servicos[0].id_usuario !== req.usuario.id) {
      return res
        .status(403)
        .json({ erro: "Você não tem permissão para deletar este serviço." });
    }

    // Deletar avaliações relacionadas primeiro (chave estrangeira)
    await banco.query("DELETE FROM oc__tb_avaliacao WHERE id_servico = ?", [
      id,
    ]);

    // Deletar serviço
    await banco.query("DELETE FROM oc__tb_servico WHERE id = ?", [id]);

    res.status(200).json({
      mensagem: "Serviço deletado com sucesso!",
    });
  } catch (erro) {
    console.error("Erro ao deletar serviço:", erro);
    res.status(500).json({ erro: "Erro ao deletar serviço." });
  }
});

module.exports = router;
```

## backend/routes/avaliacaoRoutes.js

```javascript
/**
 * Rotas de Avaliações
 * - GET /servico/:id - Listar avaliações de um serviço
 * - GET /meu-historico - Minhas avaliações (protegido)
 * - POST / - Deixar avaliação (protegido)
 * - DELETE /:id - Deletar avaliação (protegido)
 */

const express = require("express");
const banco = require("../config/database");
const { verificarToken } = require("../middlewares/autenticacao");

const router = express.Router();

// ===============================================
// 1. LISTAR AVALIAÇÕES DE UM SERVIÇO (PÚBLICO)
// ===============================================
router.get("/servico/:id", async (req, res) => {
  const { id } = req.params;

  try {
    const [avaliacoes] = await banco.query(
      `
      SELECT *
      FROM oc__vw_avaliacoes_do_servico
      WHERE id_servico = ?
      ORDER BY data_avaliacao DESC
      `,
      [id],
    );

    // Calcular média das notas
    if (avaliacoes.length > 0) {
      const somaPreco = avaliacoes.reduce((sum, a) => sum + a.nota_preco, 0);
      const somaTempo = avaliacoes.reduce(
        (sum, a) => sum + a.nota_tempo_execucao,
        0,
      );
      const somaHigiene = avaliacoes.reduce(
        (sum, a) => sum + a.nota_higiene,
        0,
      );
      const somaEducacao = avaliacoes.reduce(
        (sum, a) => sum + a.nota_educacao,
        0,
      );
      const total = avaliacoes.length;

      const medias = {
        preco: (somaPreco / total).toFixed(1),
        tempo: (somaTempo / total).toFixed(1),
        higiene: (somaHigiene / total).toFixed(1),
        educacao: (somaEducacao / total).toFixed(1),
        geral: (
          (somaPreco + somaTempo + somaHigiene + somaEducacao) /
          (total * 4)
        ).toFixed(1),
      };

      return res.status(200).json({
        total_avaliacoes: total,
        medias: medias,
        avaliacoes: avaliacoes,
      });
    }

    res.status(200).json({
      total_avaliacoes: 0,
      medias: null,
      avaliacoes: [],
    });
  } catch (erro) {
    console.error("Erro ao listar avaliações:", erro);
    res.status(500).json({ erro: "Erro ao listar avaliações." });
  }
});

// ===============================================
// 2. MINHAS AVALIAÇÕES (PROTEGIDO)
// ===============================================
router.get("/meu-historico", verificarToken, async (req, res) => {
  try {
    const [avaliacoes] = await banco.query(
      `
      SELECT *
      FROM oc__vw_historico_de_avaliacoes
      WHERE id_usuario = ?
      ORDER BY data_avaliacao DESC
      `,
      [req.usuario.id],
    );

    res.status(200).json(avaliacoes);
  } catch (erro) {
    console.error("Erro ao buscar minhas avaliações:", erro);
    res.status(500).json({ erro: "Erro ao buscar suas avaliações." });
  }
});

// ===============================================
// 3. DEIXAR AVALIAÇÃO (PROTEGIDO)
// ===============================================
router.post("/", verificarToken, async (req, res) => {
  const {
    id_servico,
    nota_preco,
    nota_tempo_execucao,
    nota_higiene,
    nota_educacao,
    comentario,
  } = req.body;

  // Validar campos
  if (
    !id_servico ||
    !nota_preco ||
    !nota_tempo_execucao ||
    !nota_higiene ||
    !nota_educacao
  ) {
    return res.status(400).json({ erro: "Todas as notas são obrigatórias!" });
  }

  // Validar se as notas estão entre 1 e 5
  const notas = [nota_preco, nota_tempo_execucao, nota_higiene, nota_educacao];
  if (notas.some((nota) => nota < 1 || nota > 5)) {
    return res.status(400).json({ erro: "As notas devem estar entre 1 e 5!" });
  }

  try {
    // Verificar se serviço existe
    const [servicos] = await banco.query(
      "SELECT * FROM oc__tb_servico WHERE id = ?",
      [id_servico],
    );

    if (servicos.length === 0) {
      return res.status(404).json({ erro: "Serviço não encontrado." });
    }

    // Verificar se o usuário é o dono do serviço
    if (servicos[0].id_usuario === req.usuario.id) {
      return res
        .status(403)
        .json({ erro: "Você não pode avaliar seu próprio serviço!" });
    }

    // Verificar se já avaliou este serviço
    const [avaliacaoExistente] = await banco.query(
      "SELECT * FROM oc__tb_avaliacao WHERE id_servico = ? AND id_usuario = ?",
      [id_servico, req.usuario.id],
    );

    if (avaliacaoExistente.length > 0) {
      return res.status(400).json({ erro: "Você já avaliou este serviço!" });
    }

    // Inserir avaliação
    const [resultado] = await banco.query(
      `INSERT INTO oc__tb_avaliacao (id_servico, id_usuario, nota_preco, nota_tempo_execucao, nota_higiene, nota_educacao, comentario, data_avaliacao)
       VALUES (?, ?, ?, ?, ?, ?, ?, NOW())`,
      [
        id_servico,
        req.usuario.id,
        nota_preco,
        nota_tempo_execucao,
        nota_higiene,
        nota_educacao,
        comentario || null,
      ],
    );

    res.status(201).json({
      mensagem: "Avaliação registrada com sucesso!",
      id_avaliacao: resultado.insertId,
    });
  } catch (erro) {
    console.error("Erro ao registrar avaliação:", erro);
    res.status(500).json({ erro: "Erro ao registrar avaliação." });
  }
});

// ===============================================
// 4. DELETAR AVALIAÇÃO (PROTEGIDO)
// ===============================================
router.delete("/:id", verificarToken, async (req, res) => {
  const { id } = req.params;

  try {
    // Verificar se existe e se é dono
    const [avaliacoes] = await banco.query(
      "SELECT * FROM oc__tb_avaliacao WHERE id = ?",
      [id],
    );

    if (avaliacoes.length === 0) {
      return res.status(404).json({ erro: "Avaliação não encontrada." });
    }

    if (avaliacoes[0].id_usuario !== req.usuario.id) {
      return res
        .status(403)
        .json({ erro: "Você não tem permissão para deletar esta avaliação." });
    }

    // Deletar avaliação
    await banco.query("DELETE FROM oc__tb_avaliacao WHERE id = ?", [id]);

    res.status(200).json({
      mensagem: "Avaliação deletada com sucesso!",
    });
  } catch (erro) {
    console.error("Erro ao deletar avaliação:", erro);
    res.status(500).json({ erro: "Erro ao deletar avaliação." });
  }
});

module.exports = router;
```

---

🎯 Resultado: criado arquivo completo com a cópia exata do código backend atual mais comentários explicativos linha a linha. 
Abra em VS Code e navegue por cada seção para dominar os fluxos de autenticação, serviços e avaliações.