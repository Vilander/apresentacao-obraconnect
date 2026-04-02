# 📚 GUIA PASSO A PASSO - 04: IMPLEMENTAR AUTENTICAÇÃO COM JWT

## 🎯 Objetivo

Implementar sistema de autenticação seguro com:
- Criptografia de senhas (bcryptjs)
- JSON Web Tokens (JWT)
- Middleware de verificação

---

## 🔐 Por que JWT?

### Sem JWT (Sessões tradicionais)

```
┌─────────┐                    ┌─────────────┐
│ Cliente │                    │  Servidor   │
│         │                    │             │
└────┬────┘                    └────┬────────┘
     │                              │
     │──── POST /login ────────→    │
     │                         Valida credenciais
     │                         Cria SESSÃO na RAM
     │                         sessionId = abc123
     │                              │
     │    ← sessionId (abc123) ──   │
     │    Salva em cookie           │
     │                              │
     │──── GET /perfil ────────→    │
     │ (cookie: abc123)         Busca sessão na RAM
     │                         Se encontra: autoriza
     │                              │
     │    ← Dados do usuário ──    │

PROBLEMA:
- Servidor precisa guardar todas sessões em RAM
- Com 10mil users = 10mil sessões em memória
- Não escala para múltiplos servidores
- Servidor cai = Todas sessões perdem
```

### Com JWT

```
┌─────────┐                    ┌─────────────┐
│ Cliente │                    │  Servidor   │
│         │                    │             │
└────┬────┘                    └────┬────────┘
     │                              │
     │──── POST /login ────────→    │
     │                         Valida credenciais
     │                         Gera JWT:
     │                         {id: 1, nome: "João"}
     │                         Assinado com SECRET
     │                              │
     │    ← JWT (eyJhbGc...) ──    │
     │    Salva em localStorage     │
     │                              │
     │──── GET /perfil ────────→    │
     │ (header: Bearer JWT)     Valida assinatura
     │                         Se válido: autoriza
     │                              │
     │    ← Dados do usuário ──    │

VANTAGEM:
- Servidor NÃO armazena sessões
- Token tem tudo que precisa
- Escala infinito (múltiplos servidores)
- Stateless (sem estado)

DESVANTAGEM:
- Token vazado = acesso até expirar
- Por isso: usar HTTPS + expiração
```

---

## 👤 Passo 1: Criar Middleware de Autenticação

**Localização:** `backend/middlewares/autenticacao.js`

```javascript
/**
 * Middleware de Autenticação com JWT
 * Verifica se o usuário está logado antes de acessar rotas protegidas
 */

const jwt = require("jsonwebtoken");

// ===============================================
// 1. MIDDLEWARE: Verificar Token JWT
// ===============================================

/**
 * Função: verificarToken
 * Descrição: Valida JWT e adiciona dados do usuário à requisição
 * 
 * Uso:
 * router.get("/perfil", verificarToken, (req, res) => {
 *   // req.usuario contém: {id, nome, email, tipo_usuario}
 * });
 */

function verificarToken(req, res, next) {
  // 1. Extrair token do header Authorization
  // Formato esperado: "Bearer eyJhbGciOi..."
  const authHeader = req.headers.authorization;
  
  if (!authHeader) {
    return res.status(401).json({
      erro: "Token não fornecido. Faça login primeiro.",
    });
  }
  
  const token = authHeader.split(" ")[1];  // Pega parte após "Bearer "
  
  if (!token) {
    return res.status(401).json({
      erro: "Formato de token inválido. Use: Bearer TOKEN",
    });
  }
  
  try {
    // 2. Verificar assinatura do token
    // jwt.verify() faz:
    // - Decodifica token
    // - Valida assinatura com JWT_SECRET
    // - Valida expiração
    
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    // 3. Adicionar dados do usuário à requisição
    // Próximas funções na rota podem usar: req.usuario
    
    req.usuario = decoded;
    
    // 4. Continuar para próxima função/rota
    next();
    
  } catch (erro) {
    // jwt.verify() lança erro em 3 casos:
    
    if (erro.name === "TokenExpiredError") {
      // Token expirou
      // Exemplo: Gerado a 7 dias atrás, supostamente por 7 dias
      return res.status(401).json({
        erro: "Token expirado. Faça login novamente.",
        expirado_em: erro.expiredAt,  // Hora que expirou
      });
    }
    
    if (erro.name === "JsonWebTokenError") {
      // Token corrompido ou assinatura inválida
      return res.status(403).json({
        erro: "Token inválido ou corrupto.",
      });
    }
    
    // Outro erro
    return res.status(403).json({
      erro: "Erro ao validar token.",
      detalhes: erro.message,
    });
  }
}

// ===============================================
// 2. MIDDLEWARE: Verificar se é Prestador
// ===============================================

/**
 * Função: verificarPrestador
 * Descrição: Apenas prestadores podem acessar
 * Deve vir DEPOIS de verificarToken
 * 
 * Uso:
 * router.post("/", verificarToken, verificarPrestador, (req, res) => {
 *   // Apenas prestadores chegam aqui
 * });
 */

function verificarPrestador(req, res, next) {
  // req.usuario vem de verificarToken
  const tipoUsuario = req.usuario.tipo_usuario;
  
  if (tipoUsuario !== "prestador" && tipoUsuario !== "admin") {
    return res.status(403).json({
      erro: "Acesso negado. Apenas prestadores podem criar serviços.",
      tipo_usuario_atual: tipoUsuario,
      solucao: "Torne-se prestador em /api/auth/tornar-prestador",
    });
  }
  
  // Continuar
  next();
}

// ===============================================
// 3. MIDDLEWARE: Verificar se é Admin
// ===============================================

function verificarAdmin(req, res, next) {
  if (req.usuario.tipo_usuario !== "admin") {
    return res.status(403).json({
      erro: "Acesso negado. Apenas administradores.",
    });
  }
  
  next();
}

// ===============================================
// EXPORTAR MIDDLEWARES
// ===============================================

module.exports = {
  verificarToken,
  verificarPrestador,
  verificarAdmin,
};
```

**Por quê essa ordem de middlewares?**

```
router.post(
  "/",
  verificarToken,           // 1º: Validar JWT
  verificarPrestador,       // 2º: Validar tipo uso
  upload.single("imagem"),  // 3º: Upload de arquivo
  async (req, res) => {...} // 4º: Lógica principal
);

E NÃO:

router.post(
  "/",
  verificarPrestador,  // ❌ ERRO! req.usuario ainda não existe
  verificarToken,
  ...
);

Fluxo:
1. Request chega
2. Passa por verificarToken → adiciona req.usuario
3. Passa por verificarPrestador → verifica req.usuario
4. Passa por upload → processa arquivo
5. Chega na rota → lógica principal
```

---

## 🔑 Passo 2: Criar Rotas de Autenticação

**Localização:** `backend/routes/authRoutes.js`

```javascript
/**
 * Rotas de Autenticação
 * - POST /registro - Registrar novo usuário
 * - POST /login - Fazer login e retornar JWT
 * - GET /perfil - Ver dados do usuário logado (protegido)
 * - PUT /tornar-prestador - Virar prestador (protegido)
 */

const express = require("express");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");
const banco = require("../config/database");
const { verificarToken } = require("../middlewares/autenticacao");

const router = express.Router();

// ===============================================
// FUNÇÃO HELPER: Gerar JWT
// ===============================================

/**
 * Função genérica para gerar tokens
 * Reutilizável em login, registro, etc
 */

function gerarToken(usuario) {
  return jwt.sign(
    {
      // PAYLOAD: Dados que serão decodificados
      id: usuario.id,
      nome: usuario.nome_usuario,
      email: usuario.email,
      tipo_usuario: usuario.tipo_usuario,
    },
    // SECRET: Chave privada para assinar
    process.env.JWT_SECRET,
    // OPTIONS
    {
      expiresIn: process.env.JWT_EXPIRY,  // "7d"
    },
  );
}

// ===============================================
// 1. POST /registro - Registrar novo usuário
// ===============================================

/**
 * Endpoint: POST /api/auth/registro
 * 
 * Body:
 * {
 *   "nome_usuario": "João Silva",
 *   "email": "joao@email.com",
 *   "senha": "senha123",
 *   "login": "joao_silva",
 *   "telefone": "11999999999"  // opcional
 * }
 * 
 * Response: 201
 * {
 *   "mensagem": "Usuário registrado com sucesso!",
 *   "id_usuario": 1,
 *   "usuario": "João Silva"
 * }
 */

router.post("/registro", async (req, res) => {
  try {
    const { nome_usuario, email, senha, login, telefone } = req.body;
    
    // ───────────────────────────────────────────
    // VALIDAÇÕES (Frontend também valida, mas é bom repetir)
    // ───────────────────────────────────────────
    
    // 1. Campos obrigatórios
    if (!nome_usuario || !email || !senha || !login) {
      return res.status(400).json({
        erro: "Todos os campos são obrigatórios!",
        campos_necessarios: [
          "nome_usuario",
          "email",
          "senha",
          "login",
        ],
      });
    }
    
    // 2. Validar email com regex
    const regexEmail = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!regexEmail.test(email)) {
      return res.status(400).json({
        erro: "Email inválido!",
        exemplo: "usuario@email.com",
      });
    }
    
    // 3. Validar comprimento de senha
    if (senha.length < 6) {
      return res.status(400).json({
        erro: "Senha deve ter pelo menos 6 caracteres!",
        comprimento_minimo: 6,
        comprimento_fornecido: senha.length,
      });
    }
    
    // ───────────────────────────────────────────
    // VERIFICAR DUPLICATAS
    // ───────────────────────────────────────────
    
    // SQL com placeholders (?) previne SQL injection
    const [usuariosExistentes] = await banco.query(
      "SELECT * FROM oc__tb_usuario WHERE email = ? OR login = ?",
      [email, login],  // Valores substituem ?
    );
    
    if (usuariosExistentes.length > 0) {
      return res.status(409).json({
        erro: "E-mail ou Login já cadastrados!",
        campo_problema: usuariosExistentes[0].email === email ? "email" : "login",
      });
    }
    
    // ───────────────────────────────────────────
    // CRIPTOGRAFAR SENHA COM BCRYPTJS
    // ───────────────────────────────────────────
    
    /**
     * bcryptjs.genSalt(rounds)
     * rounds: Quantas vezes embaralhar (padrão 10)
     * Quanto maior = mais seguro mas mais lento
     * 
     * Tempo aproximado:
     * 8 rounds = 80ms
     * 10 rounds = 200ms (recomendado)
     * 12 rounds = 1s
     */
    
    const salt = await bcrypt.genSalt(10);
    
    // bcryptjs.hash(senha, salt)
    // Retorna: hash criptografado
    // Mesmo com salt, senhas iguais geram hashes DIFERENTES
    
    const senhaCriptografada = await bcrypt.hash(senha, salt);
    
    // ───────────────────────────────────────────
    // INSERIR USUÁRIO NO BANCO
    // ───────────────────────────────────────────
    
    const [resultado] = await banco.query(
      `INSERT INTO oc__tb_usuario 
       (nome_usuario, email, senha, login, telefone, tipo_usuario, data_cadastro) 
       VALUES (?, ?, ?, ?, ?, ?, NOW())`,
      [
        nome_usuario,
        email,
        senhaCriptografada,  // Salvar Hash, não a senha
        login,
        telefone || null,    // null se não fornecer
        "usuario",           // Tipo padrão
      ],
    );
    
    // resultado.insertId = ID auto-gerado pelo banco
    
    res.status(201).json({
      mensagem: "Usuário registrado com sucesso!",
      id_usuario: resultado.insertId,
      usuario: nome_usuario,
      proximo_passo: "Faça login em /api/auth/login",
    });
    
  } catch (erro) {
    console.error("❌ Erro ao registrar usuário:", erro);
    res.status(500).json({
      erro: "Erro ao registrar usuário.",
      detalhes: erro.message,
    });
  }
});

// ===============================================
// 2. POST /login - Fazer login e retornar JWT
// ===============================================

/**
 * Endpoint: POST /api/auth/login
 * 
 * Body:
 * {
 *   "login": "joao_silva",  // ou "joao@email.com"
 *   "senha": "senha123"
 * }
 * 
 * Response: 200
 * {
 *   "mensagem": "Login realizado com sucesso!",
 *   "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
 *   "usuario": {
 *     "id": 1,
 *     "nome": "João Silva",
 *     "email": "joao@email.com",
 *     "tipo_usuario": "prestador"
 *   }
 * }
 */

router.post("/login", async (req, res) => {
  try {
    const { login, senha } = req.body;
    
    // Validar campos
    if (!login || !senha) {
      return res.status(400).json({
        erro: "Login e senha são obrigatórios!",
      });
    }
    
    // ───────────────────────────────────────────
    // BUSCAR USUÁRIO POR LOGIN OU EMAIL
    // ───────────────────────────────────────────
    
    const [usuarios] = await banco.query(
      "SELECT * FROM oc__tb_usuario WHERE login = ? OR email = ?",
      [login, login],  // Se é email, pega por email; se é login, pega por login
    );
    
    // Usuário não existe
    if (usuarios.length === 0) {
      return res.status(401).json({
        erro: "Usuário ou senha incorretos.",
        dica: "Verifique login/email e tente novamente",
      });
    }
    
    const usuario = usuarios[0];
    
    // ───────────────────────────────────────────
    // COMPARAR SENHAS COM BCRYPTJS
    // ───────────────────────────────────────────
    
    /**
     * bcryptjs.compare(senhaFornecida, hashArmazenado)
     * Retorna: true ou false
     * 
     * Como funciona:
     * 1. Aplica mesmas transformações à senha fornecida
     * 2. Compara com hash armazenado
     * 3. Retorna true se combinam
     * 
     * Propriedade: Irreversível
     * - Não consegue recuperar senha original (bom!)
     * - Só consegue comparar senhas
     */
    
    const senhaValida = await bcrypt.compare(senha, usuario.senha);
    
    if (!senhaValida) {
      return res.status(401).json({
        erro: "Usuário ou senha incorretos.",
      });
    }
    
    // ───────────────────────────────────────────
    // GERAR E RETORNAR JWT
    // ───────────────────────────────────────────
    
    const token = gerarToken(usuario);
    
    res.status(200).json({
      mensagem: "Login realizado com sucesso!",
      token: token,
      usuario: {
        id: usuario.id,
        nome: usuario.nome_usuario,
        email: usuario.email,
        tipo_usuario: usuario.tipo_usuario,
      },
      expira_em: process.env.JWT_EXPIRY,
    });
    
  } catch (erro) {
    console.error("❌ Erro ao fazer login:", erro);
    res.status(500).json({
      erro: "Erro ao fazer login.",
      detalhes: erro.message,
    });
  }
});

// ===============================================
// 3. GET /perfil - Ver dados do usuário (PROTEGIDO)
// ===============================================

/**
 * Endpoint: GET /api/auth/perfil
 * Autenticação: REQUERIDA (Bearer TOKEN)
 * 
 * Headers:
 * {
 *   "Authorization": "Bearer eyJhbGciOi..."
 * }
 * 
 * Response: 200
 * {
 *   "id": 1,
 *   "nome_usuario": "João Silva",
 *   "email": "joao@email.com",
 *   "login": "joao_silva",
 *   "telefone": "11999999999",
 *   "tipo_usuario": "prestador",
 *   "data_cadastro": "2026-01-22T19:08:35.000Z"
 * }
 */

router.get("/perfil", verificarToken, async (req, res) => {
  try {
    // req.usuario vem do middleware verificarToken
    // Contém: id, nome, email, tipo_usuario
    
    // Buscar dados atualizados do banco (não confiar só no token)
    const [usuarios] = await banco.query(
      "SELECT * FROM oc__tb_usuario WHERE id = ?",
      [req.usuario.id],
    );
    
    if (usuarios.length === 0) {
      return res.status(404).json({
        erro: "Usuário não encontrado.",
      });
    }
    
    const usuario = usuarios[0];
    
    // NÃO retornar senha
    delete usuario.senha;
    
    res.status(200).json(usuario);
    
  } catch (erro) {
    console.error("❌ Erro ao buscar perfil:", erro);
    res.status(500).json({
      erro: "Erro ao buscar perfil.",
      detalhes: erro.message,
    });
  }
});

// ===============================================
// 4. PUT /tornar-prestador - Virar prestador (PROTEGIDO)
// ===============================================

/**
 * Endpoint: PUT /api/auth/tornar-prestador
 * Autenticação: REQUERIDA
 * 
 * Body: {} (vazio)
 * 
 * Response: 200
 * {
 *   "mensagem": "Você agora é um prestador!",
 *   "tipo_usuario": "prestador"
 * }
 */

router.put("/tornar-prestador", verificarToken, async (req, res) => {
  try {
    const userId = req.usuario.id;
    
    // Validar se já é prestador
    const [usuarios] = await banco.query(
      "SELECT tipo_usuario FROM oc__tb_usuario WHERE id = ?",
      [userId],
    );
    
    if (usuarios.length === 0) {
      return res.status(404).json({
        erro: "Usuário não encontrado.",
      });
    }
    
    const tipoAtual = usuarios[0].tipo_usuario;
    
    if (tipoAtual === "prestador") {
      return res.status(400).json({
        erro: "Você já é um prestador!",
        tipo_usuario: tipoAtual,
      });
    }
    
    // UPDATE
    await banco.query(
      "UPDATE oc__tb_usuario SET tipo_usuario = ? WHERE id = ?",
      ["prestador", userId],
    );
    
    res.status(200).json({
      mensagem: "Você agora é um prestador!",
      tipo_usuario: "prestador",
      pode_criar_servicos: true,
    });
    
  } catch (erro) {
    console.error("❌ Erro ao tornar prestador:", erro);
    res.status(500).json({
      erro: "Erro ao atualizar tipo de usuário.",
      detalhes: erro.message,
    });
  }
});

// ===============================================
// EXPORTAR ROUTER
// ===============================================

module.exports = router;
```

---

## ✅ Checklist

- [ ] Arquivo `backend/middlewares/autenticacao.js` criado
- [ ] Arquivo `backend/routes/authRoutes.js` criado
- [ ] JWT_SECRET definido em `.env`
- [ ] JWT_EXPIRY definido em `.env`
- [ ] Rotas de auth funcionando

---

## 🧪 Testar Autenticação

**1. Registrar usuário:**

```bash
POST http://localhost:3001/api/auth/registro
Content-Type: application/json

{
  "nome_usuario": "João Silva",
  "email": "joao@email.com",
  "senha": "senha123",
  "login": "joao_silva",
  "telefone": "11999999999"
}

# Response: 201
# {
#   "mensagem": "Usuário registrado com sucesso!",
#   "id_usuario": 1,
#   "usuario": "João Silva"
# }
```

**2. Fazer login:**

```bash
POST http://localhost:3001/api/auth/login
Content-Type: application/json

{
  "login": "joao_silva",
  "senha": "senha123"
}

# Response: 200
# {
#   "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
#   "usuario": {...}
# }

# COPIAR TOKEN (será usado em requisições protegidas)
```

**3. Ver perfil (protegido):**

```bash
GET http://localhost:3001/api/auth/perfil
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Response: 200 with user data
```

---

✅ **Autenticação implementada!** Próximo passo: Implementar rotas de serviços.
