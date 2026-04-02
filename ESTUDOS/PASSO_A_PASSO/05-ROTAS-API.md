# 📚 GUIA PASSO A PASSO - 05: ROTAS DE SERVIÇOS E AVALIAÇÕES

## 🎯 Objetivo

Implementar CRUD completo de serviços e sistema de avaliações

---

## 🛠️ Passo 1: Criar Rotas de Serviços

**Localização:** `backend/routes/servicoRoutes.js`

```javascript
/**
 * Rotas de Serviços
 * - GET / - Listar todos (público)
 * - GET /:id - Detalhes (público)
 * - GET /meus/servicos - Meus serviços (protegido)
 * - POST / - Criar (protegido + prestador)
 * - PUT /:id - Editar (protegido + prestador)
 * - DELETE /:id - Deletar (protegido + prestador)
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
// 1. GET / - Listar todos os serviços (PÚBLICO)
// ===============================================

/**
 * Endpoint: GET /api/servicos
 * 
 * Autenticação: NÃO REQUERIDA
 * 
 * Query complexa com JOINs e agregação:
 * - Buscar serviço
 * - Fazer JOIN com usuário (prestador)
 * - LEFT JOIN com avaliações
 * - Calcular nota média
 * - Contar total de avaliações
 * 
 * Response: 200
 * [
 *   {
 *     "id": 1,
 *     "titulo": "Encanamento",
 *     "desc_servico": "...",
 *     "nome_usuario": "João",
 *     "email": "joao@email.com",
 *     "nota_media": 4.75,
 *     "total_avaliacoes": 4
 *   }
 * ]
 */

router.get("/", async (req, res) => {
  try {
    // Query com JOINs para dados completos
    const [servicos] = await banco.query(`
      SELECT 
        s.*,
        u.nome_usuario,
        u.email,
        CAST(COALESCE(AVG((a.nota_preco + a.nota_tempo_execucao + a.nota_higiene + a.nota_educacao) / 4), 0) AS DECIMAL(10,2)) as nota_media,
        COUNT(a.id) as total_avaliacoes
      FROM oc__tb_servico s
      JOIN oc__tb_usuario u ON s.id_usuario = u.id
      LEFT JOIN oc__tb_avaliacao a ON s.id = a.id_servico
      GROUP BY s.id
      ORDER BY s.data_cadastro DESC
    `);

    // Converter nota_media string para número
    const servicosFormatados = servicos.map((s) => ({
      ...s,
      nota_media: parseFloat(s.nota_media) || 0,
    }));

    res.status(200).json(servicosFormatados);
  } catch (erro) {
    console.error("❌ Erro ao listar serviços:", erro);
    res.status(500).json({ erro: "Erro ao listar serviços." });
  }
});

// ===============================================
// 2. GET /meus/servicos - Meus serviços (PROTEGIDO)
// ===============================================

/**
 * IMPORTANTE: Esta rota DEVE vir ANTES de GET /:id
 * Se não, ficaria:
 * GET /meus/servicos seria interpretado como GET /:id com id="meus"
 * 
 * Ordem correta:
 * 1º GET /meus/servicos
 * 2º GET /:id
 */

router.get("/meus/servicos", verificarToken, async (req, res) => {
  try {
    const [servicos] = await banco.query(
      "SELECT * FROM oc__tb_servico WHERE id_usuario = ? ORDER BY data_cadastro DESC",
      [req.usuario.id],
    );

    res.status(200).json(servicos);
  } catch (erro) {
    console.error("❌ Erro ao listar meus serviços:", erro);
    res.status(500).json({ erro: "Erro ao listar seus serviços." });
  }
});

// ===============================================
// 3. GET /:id - Buscar serviço por ID (PÚBLICO)
// ===============================================

/**
 * Endpoint: GET /api/servicos/1
 * 
 * Response: 200
 * {
 *   "id": 1,
 *   "id_usuario": 1,
 *   "titulo": "Encanamento",
 *   "desc_servico": "...",
 *   "nome_usuario": "João",
 *   "telefone": "11999999999",
 *   "nota_media": 4.75,
 *   "total_avaliacoes": 4
 * }
 */

router.get("/:id", async (req, res) => {
  const { id } = req.params;

  try {
    const [servicos] = await banco.query(
      `
      SELECT 
        s.*,
        u.nome_usuario,
        u.email,
        u.telefone,
        CAST(COALESCE(AVG((a.nota_preco + a.nota_tempo_execucao + a.nota_higiene + a.nota_educacao) / 4), 0) AS DECIMAL(10,2)) as nota_media,
        COUNT(a.id) as total_avaliacoes
      FROM oc__tb_servico s
      JOIN oc__tb_usuario u ON s.id_usuario = u.id
      LEFT JOIN oc__tb_avaliacao a ON s.id = a.id_servico
      WHERE s.id = ?
      GROUP BY s.id
    `,
      [id],
    );

    if (servicos.length === 0) {
      return res.status(404).json({ erro: "Serviço não encontrado." });
    }

    const servico = servicos[0];
    servico.nota_media = parseFloat(servico.nota_media) || 0;

    res.status(200).json(servico);
  } catch (erro) {
    console.error("❌ Erro ao buscar serviço:", erro);
    res.status(500).json({ erro: "Erro ao buscar serviço." });
  }
});

// ===============================================
// 4. POST / - Criar novo serviço (PROTEGIDO + PRESTADOR)
// ===============================================

/**
 * Endpoint: POST /api/servicos
 * 
 * Autenticação: REQUERIDA
 * Tipo: PRESTADOR
 * Content-Type: multipart/form-data
 * 
 * Body (FormData):
 * {
 *   "titulo": "Encanamento",
 *   "descricao": "Conserto de vazamentos",
 *   "id_categoria": 4,
 *   "imagem": <arquivo.jpg>
 * }
 * 
 * Response: 201
 * {
 *   "mensagem": "Serviço criado com sucesso!",
 *   "id_servico": 5,
 *   "imagem": "http://localhost:3001/uploads/imagem-123123.jpg"
 * }
 */

router.post(
  "/",
  verificarToken,              // 1º: Validar JWT
  verificarPrestador,          // 2º: Validar tipo usuário
  upload.single("imagem"),     // 3º: Upload de arquivo
  async (req, res) => {
    const { titulo, descricao, id_categoria } = req.body;

    // Validar campos
    if (!titulo || !descricao) {
      return res.status(400).json({
        erro: "Título e Descrição são obrigatórios!",
      });
    }

    try {
      // Montar URL da imagem
      let imagemUrl = null;
      if (req.file) {
        // req.file.filename = "imagem-123123.jpg"
        imagemUrl = `http://localhost:${process.env.PORT}/uploads/${req.file.filename}`;
      }

      // Inserir no banco
      const [resultado] = await banco.query(
        `INSERT INTO oc__tb_servico 
         (id_usuario, titulo, desc_servico, id_categoria, imagem_url, data_cadastro) 
         VALUES (?, ?, ?, ?, ?, NOW())`,
        [
          req.usuario.id,      // ID do prestador logado
          titulo,
          descricao,
          id_categoria || null,
          imagemUrl,
        ],
      );

      res.status(201).json({
        mensagem: "Serviço criado com sucesso!",
        id_servico: resultado.insertId,
        imagem: imagemUrl,
      });
    } catch (erro) {
      console.error("❌ Erro ao criar serviço:", erro);
      res.status(500).json({ erro: "Erro ao criar serviço." });
    }
  },
);

// ===============================================
// 5. PUT /:id - Editar serviço (PROTEGIDO + PRESTADOR)
// ===============================================

/**
 * Endpoint: PUT /api/servicos/1
 * 
 * Autenticação: REQUERIDA
 * Tipo: PRESTADOR
 * 
 * Body:
 * {
 *   "titulo": "Novo título",
 *   "descricao": "Nova descrição",
 *   "id_categoria": 5
 * }
 * 
 * Validações:
 * - Serviço deve existir
 * - User deve ser criador
 * 
 * Response: 200
 * {
 *   "mensagem": "Serviço atualizado com sucesso!"
 * }
 */

router.put("/:id", verificarToken, verificarPrestador, async (req, res) => {
  const { id } = req.params;
  const { titulo, descricao, id_categoria } = req.body;

  try {
    // Validar se serviço existe e pertence ao user
    const [servicos] = await banco.query(
      "SELECT * FROM oc__tb_servico WHERE id = ? AND id_usuario = ?",
      [id, req.usuario.id],
    );

    if (servicos.length === 0) {
      return res.status(403).json({
        erro: "Serviço não encontrado ou você não tem permissão!",
      });
    }

    // Atualizar
    await banco.query(
      `UPDATE oc__tb_servico 
       SET titulo = COALESCE(?, titulo),
           desc_servico = COALESCE(?, desc_servico),
           id_categoria = COALESCE(?, id_categoria)
       WHERE id = ?`,
      [
        titulo || null,
        descricao || null,
        id_categoria || null,
        id,
      ],
    );

    res.status(200).json({
      mensagem: "Serviço atualizado com sucesso!",
    });
  } catch (erro) {
    console.error("❌ Erro ao atualizar serviço:", erro);
    res.status(500).json({ erro: "Erro ao atualizar serviço." });
  }
});

// ===============================================
// 6. DELETE /:id - Deletar serviço (PROTEGIDO + PRESTADOR)
// ===============================================

/**
 * Endpoint: DELETE /api/servicos/1
 * 
 * Autenticação: REQUERIDA
 * Tipo: PRESTADOR
 * 
 * Response: 200
 * {
 *   "mensagem": "Serviço deletado com sucesso!"
 * }
 */

router.delete("/:id", verificarToken, verificarPrestador, async (req, res) => {
  const { id } = req.params;

  try {
    // Validar se serviço existe e pertence ao user
    const [servicos] = await banco.query(
      "SELECT * FROM oc__tb_servico WHERE id = ? AND id_usuario = ?",
      [id, req.usuario.id],
    );

    if (servicos.length === 0) {
      return res.status(403).json({
        erro: "Serviço não encontrado ou você não tem permissão!",
      });
    }

    // Deletar (avaliações deletam automaticamente por CASCADE)
    await banco.query(
      "DELETE FROM oc__tb_servico WHERE id = ?",
      [id],
    );

    res.status(200).json({
      mensagem: "Serviço deletado com sucesso!",
    });
  } catch (erro) {
    console.error("❌ Erro ao deletar serviço:", erro);
    res.status(500).json({ erro: "Erro ao deletar serviço." });
  }
});

// ===============================================
// EXPORTAR
// ===============================================

module.exports = router;
```

---

## ⭐ Passo 2: Criar Rotas de Avaliações

**Localização:** `backend/routes/avaliacaoRoutes.js`

```javascript
/**
 * Rotas de Avaliações
 * - GET /servico/:id - Listar avaliações de um serviço (público)
 * - GET /meu-historico - Minhas avaliações (protegido)
 * - POST / - Deixar avaliação (protegido)
 * - DELETE /:id - Deletar avaliação (protegido)
 */

const express = require("express");
const banco = require("../config/database");
const { verificarToken } = require("../middlewares/autenticacao");

const router = express.Router();

// ===============================================
// 1. GET /servico/:id - Listar avaliações (PÚBLICO)
// ===============================================

/**
 * Endpoint: GET /api/avaliacoes/servico/1
 * 
 * Response: 200
 * {
 *   "total_avaliacoes": 4,
 *   "medias": {
 *     "preco": 4.8,
 *     "tempo": 4.5,
 *     "higiene": 4.8,
 *     "educacao": 4.8,
 *     "geral": 4.73
 *   },
 *   "avaliacoes": [
 *     {
 *       "id": 1,
 *       "id_servico": 1,
 *       "nota_preco": 5,
 *       "nota_tempo_execucao": 4,
 *       "nota_higiene": 5,
 *       "nota_educacao": 5,
 *       "comentario": "Excelente!",
 *       "nome_usuario": "Maria"
 *     }
 *   ]
 * }
 */

router.get("/servico/:id", async (req, res) => {
  const { id } = req.params;

  try {
    const [avaliacoes] = await banco.query(
      `
      SELECT a.*, u.nome_usuario
      FROM oc__tb_avaliacao a
      JOIN oc__tb_usuario u ON a.id_usuario = u.id
      WHERE a.id_servico = ?
      ORDER BY a.data_avaliacao DESC
    `,
      [id],
    );

    // Calcular médias se houver avaliações
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

    // Sem avaliações
    res.status(200).json({
      total_avaliacoes: 0,
      medias: null,
      avaliacoes: [],
    });
  } catch (erro) {
    console.error("❌ Erro ao listar avaliações:", erro);
    res.status(500).json({ erro: "Erro ao listar avaliações." });
  }
});

// ===============================================
// 2. GET /meu-historico - Minhas avaliações (PROTEGIDO)
// ===============================================

/**
 * Endpoint: GET /api/avaliacoes/meu-historico
 * 
 * Autenticação: REQUERIDA
 * 
 * Response: 200
 * [
 *   {
 *     "id": 1,
 *     "id_servico": 1,
 *     "nota_preco": 5,
 *     "nome_servico": "Encanamento",
 *     "nome_usuario": "João (prestador)"
 *   }
 * ]
 */

router.get("/meu-historico", verificarToken, async (req, res) => {
  try {
    const [avaliacoes] = await banco.query(
      `
      SELECT a.*, s.titulo as nome_servico, u.nome_usuario
      FROM oc__tb_avaliacao a
      JOIN oc__tb_servico s ON a.id_servico = s.id
      JOIN oc__tb_usuario u ON s.id_usuario = u.id
      WHERE a.id_usuario = ?
      ORDER BY a.data_avaliacao DESC
    `,
      [req.usuario.id],
    );

    res.status(200).json(avaliacoes);
  } catch (erro) {
    console.error("❌ Erro ao buscar minhas avaliações:", erro);
    res.status(500).json({ erro: "Erro ao buscar suas avaliações." });
  }
});

// ===============================================
// 3. POST / - Criar avaliação (PROTEGIDO)
// ===============================================

/**
 * Endpoint: POST /api/avaliacoes
 * 
 * Autenticação: REQUERIDA
 * 
 * Body:
 * {
 *   "id_servico": 1,
 *   "nota_preco": 5,
 *   "nota_tempo_execucao": 4,
 *   "nota_higiene": 5,
 *   "nota_educacao": 5,
 *   "comentario": "Excelente profissional!"
 * }
 * 
 * Validações:
 * - Notas entre 1-5
 * - Serviço existe
 * - Não avaliar próprio serviço
 * 
 * Response: 201
 * {
 *   "mensagem": "Avaliação registrada com sucesso!",
 *   "id_avaliacao": 5
 * }
 */

router.post("/", verificarToken, async (req, res) => {
  try {
    const {
      id_servico,
      nota_preco,
      nota_tempo_execucao,
      nota_higiene,
      nota_educacao,
      comentario,
    } = req.body;

    // ───────────────────────────────────────────
    // VALIDAÇÕES
    // ───────────────────────────────────────────

    // 1. Campos obrigatórios
    if (!id_servico) {
      return res.status(400).json({
        erro: "ID do serviço é obrigatório!",
      });
    }

    // 2. Notas devem estar entre 1-5
    const notas = [nota_preco, nota_tempo_execucao, nota_higiene, nota_educacao];
    if (notas.some((nota) => nota < 1 || nota > 5)) {
      return res.status(400).json({
        erro: "Notas devem estar entre 1 e 5!",
      });
    }

    // 3. Verificar se serviço existe
    const [servicos] = await banco.query(
      "SELECT id_usuario FROM oc__tb_servico WHERE id = ?",
      [id_servico],
    );

    if (servicos.length === 0) {
      return res.status(404).json({
        erro: "Serviço não encontrado!",
      });
    }

    // 4. Validar: não avaliar próprio serviço
    if (servicos[0].id_usuario === req.usuario.id) {
      return res.status(403).json({
        erro: "Você não pode avaliar seu próprio serviço!",
      });
    }

    // ───────────────────────────────────────────
    // INSERIR AVALIAÇÃO
    // ───────────────────────────────────────────

    const [resultado] = await banco.query(
      `INSERT INTO oc__tb_avaliacao 
       (id_servico, id_usuario, nota_preco, nota_tempo_execucao, nota_higiene, nota_educacao, comentario, data_avaliacao) 
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
    console.error("❌ Erro ao criar avaliação:", erro);
    res.status(500).json({ erro: "Erro ao registrar avaliação." });
  }
});

// ===============================================
// 4. DELETE /:id - Deletar avaliação (PROTEGIDO)
// ===============================================

/**
 * Endpoint: DELETE /api/avaliacoes/1
 * 
 * Autenticação: REQUERIDA
 * 
 * Validações:
 * - Avaliação deve existir
 * - Deve ser criador
 * 
 * Response: 200
 * {
 *   "mensagem": "Avaliação deletada com sucesso!"
 * }
 */

router.delete("/:id", verificarToken, async (req, res) => {
  const { id } = req.params;

  try {
    // Verificar se avaliação existe e pertence ao user
    const [avaliacoes] = await banco.query(
      "SELECT * FROM oc__tb_avaliacao WHERE id = ? AND id_usuario = ?",
      [id, req.usuario.id],
    );

    if (avaliacoes.length === 0) {
      return res.status(403).json({
        erro: "Avaliação não encontrada ou você não tem permissão!",
      });
    }

    // Deletar
    await banco.query(
      "DELETE FROM oc__tb_avaliacao WHERE id = ?",
      [id],
    );

    res.status(200).json({
      mensagem: "Avaliação deletada com sucesso!",
    });
  } catch (erro) {
    console.error("❌ Erro ao deletar avaliação:", erro);
    res.status(500).json({ erro: "Erro ao deletar avaliação." });
  }
});

// ===============================================
// EXPORTAR
// ===============================================

module.exports = router;
```

---

## 🔄 Passo 3: Registrar Rotas no index.js

Em `backend/index.js`, descomentar e adicionar:

```javascript
// Importar rotas
const authRoutes = require("./routes/authRoutes");
const servicoRoutes = require("./routes/servicoRoutes");
const avaliacaoRoutes = require("./routes/avaliacaoRoutes");

// ... middlewares ...

// Registrar rotas
app.use("/api/auth", authRoutes);
app.use("/api/servicos", servicoRoutes);
app.use("/api/avaliacoes", avaliacaoRoutes);
```

---

## ✅ Checklist

- [ ] Arquivo `backend/routes/servicoRoutes.js` criado
- [ ] Arquivo `backend/routes/avaliacaoRoutes.js` criado
- [ ] Rotas registradas em `index.js`
- [ ] Servidor testado com `npm start`

---

## 🧪 Testar Rotas

**1. Listar todos os serviços:**
```bash
GET http://localhost:3001/api/servicos
```

**2. Criar novo serviço (FormData com imagem):**
```bash
POST http://localhost:3001/api/servicos
Authorization: Bearer TOKEN
Content-Type: multipart/form-data

Fields:
- titulo: "Encanamento"
- descricao: "..."
- imagem: <arquivo.jpg>
```

**3. Criar avaliação:**
```bash
POST http://localhost:3001/api/avaliacoes
Authorization: Bearer TOKEN

{
  "id_servico": 1,
  "nota_preco": 5,
  "nota_tempo_execucao": 4,
  "nota_higiene": 5,
  "nota_educacao": 5,
  "comentario": "Ótimo!"
}
```

---

✅ **Backend completo!** Próximo passo: Criar frontend.
