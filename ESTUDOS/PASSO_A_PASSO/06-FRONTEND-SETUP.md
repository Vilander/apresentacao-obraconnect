# 📚 GUIA PASSO A PASSO - 06: ESTRUTURA DO FRONTEND

## 🎯 Objetivo

Criar a interface web SPA com HTML, CSS e JavaScript Vanilla

---

## 📁 Passo 1: Estrutura de Pastas Frontend

```
frontend/
├── index.html                  # Página principal
├── style.css                   # Estilos globais
├── assets/
│   └── imgs/                  # Imagens do projeto
├── js/
│   ├── api.js                 # Requisições HTTP
│   ├── auth.js                # Autenticação
│   └── servicos.js            # Gerenciamento serviços
└── pages/
    ├── login.html             # Tela de login
    ├── registro.html          # Tela de registro
    ├── meu-perfil.html        # Perfil do usuário
    ├── cadastrar-servico.html # Criar serviço
    └── detalhes-servico.html  # Detalhes e avaliações
```

---

## 🟢 Passo 2: Criar `js/api.js` - Camada HTTP

Este arquivo centraliza todas as requisições à API:

```javascript
/**
 * API Helper - ObraConnect
 * Funções reutilizáveis para fazer requisições à API
 */

// URL base da API
const API_BASE_URL = "http://localhost:3001/api";

// ===============================================
// FUNÇÃO GENÉRICA: Fazer Requisição HTTP
// ===============================================

/**
 * Função: fazerRequisicao
 * Objetivo: Abstrair lógica repetitiva de fetch
 * 
 * Parâmetros:
 * - endpoint: "/auth/login"
 * - metodo: "GET", "POST", "PUT", "DELETE"
 * - dados: corpo JSON (opcional)
 * 
 * Retorna:
 * - {sucesso: true, dados: {...}} se OK
 * - {sucesso: false, erro: "mensagem"} se erro
 */

async function fazerRequisicao(endpoint, metodo = "GET", dados = null) {
  try {
    const opcoes = {
      method: metodo,
      headers: {
        "Content-Type": "application/json",
      },
    };

    // Adicionar token do localStorage se existir
    const token = localStorage.getItem("token");
    if (token) {
      opcoes.headers["Authorization"] = `Bearer ${token}`;
    }

    // Adicionar corpo da requisição se existir
    if (dados) {
      opcoes.body = JSON.stringify(dados);
    }

    // Fazer requisição
    const resposta = await fetch(`${API_BASE_URL}${endpoint}`, opcoes);
    const resultado = await resposta.json();

    // Se não foi 200-299, throw erro
    if (!resposta.ok) {
      throw new Error(resultado.erro || "Erro na requisição");
    }

    return {
      sucesso: true,
      dados: resultado,
    };
  } catch (erro) {
    return {
      sucesso: false,
      erro: erro.message,
    };
  }
}

// ===============================================
// FUNÇÕES DE AUTENTICAÇÃO
// ===============================================

async function registrarUsuario(nome, email, senha, login, telefone) {
  return fazerRequisicao("/auth/registro", "POST", {
    nome_usuario: nome,
    email,
    senha,
    login,
    telefone: telefone || null,
  });
}

async function fazerLogin(login, senha) {
  return fazerRequisicao("/auth/login", "POST", { login, senha });
}

async function verPerfil() {
  return fazerRequisicao("/auth/perfil", "GET");
}

async function tornarPrestador() {
  return fazerRequisicao("/auth/tornar-prestador", "PUT");
}

// ===============================================
// FUNÇÕES DE SERVIÇOS
// ===============================================

async function listarServicos() {
  return fazerRequisicao("/servicos", "GET");
}

async function buscarServicoPorId(id) {
  return fazerRequisicao(`/servicos/${id}`, "GET");
}

async function listarMeusServicos() {
  return fazerRequisicao("/servicos/meus/servicos", "GET");
}

async function atualizarServico(id, titulo, descricao, id_categoria) {
  return fazerRequisicao(`/servicos/${id}`, "PUT", {
    titulo,
    descricao,
    id_categoria,
  });
}

async function deletarServico(id) {
  return fazerRequisicao(`/servicos/${id}`, "DELETE");
}

// ===============================================
// FUNÇÕES DE AVALIAÇÕES
// ===============================================

async function listarAvaliacoes(id_servico) {
  return fazerRequisicao(`/avaliacoes/servico/${id_servico}`, "GET");
}

async function listarMinhasAvaliacoes() {
  return fazerRequisicao("/avaliacoes/meu-historico", "GET");
}

async function criarAvaliacao(
  id_servico,
  nota_preco,
  nota_tempo,
  nota_higiene,
  nota_educacao,
  comentario
) {
  return fazerRequisicao("/avaliacoes", "POST", {
    id_servico,
    nota_preco,
    nota_tempo_execucao: nota_tempo,
    nota_higiene,
    nota_educacao,
    comentario,
  });
}

async function deletarAvaliacao(id) {
  return fazerRequisicao(`/avaliacoes/${id}`, "DELETE");
}
```

---

## 🟢 Passo 3: Criar `js/auth.js` - Gerenciar Autenticação

```javascript
/**
 * Autenticação - ObraConnect
 * Gerencia login, logout e estado de autenticação
 */

// Estado global
let usuarioLogado = null;
let tokenAtual = null;

// ===============================================
// Inicializar ao carregar página
// ===============================================

function inicializarAutenticacao() {
  const token = localStorage.getItem("token");
  const usuario = localStorage.getItem("usuario");

  if (token && usuario) {
    try {
      usuarioLogado = JSON.parse(usuario);
      tokenAtual = token;
      atualizarUIAutenticacao();
    } catch (erro) {
      console.error("Erro ao restaurar autenticação:", erro);
      fazerLogout();
    }
  }
}

// ===============================================
// Realizar Login
// ===============================================

async function realizarLogin(loginOuEmail, senha) {
  try {
    const resultado = await fazerLogin(loginOuEmail, senha);

    if (!resultado.sucesso) {
      return {
        sucesso: false,
        erro: resultado.erro,
      };
    }

    // Guardar no localStorage
    const { token, usuario } = resultado.dados;
    localStorage.setItem("token", token);
    localStorage.setItem("usuario", JSON.stringify(usuario));

    // Atualizar estado global
    usuarioLogado = usuario;
    tokenAtual = token;

    // Atualizar UI
    atualizarUIAutenticacao();

    // Redirecionar
    setTimeout(() => {
      window.location.href = "../index.html"; // ou a página desejada
    }, 500);

    return {
      sucesso: true,
      usuario,
    };
  } catch (erro) {
    console.error("Erro ao fazer login:", erro);
    return {
      sucesso: false,
      erro: "Erro ao fazer login",
    };
  }
}

// ===============================================
// Realizar Registro
// ===============================================

async function realizarRegistro(nome, email, senha, login, telefone) {
  try {
    // Validações no frontend
    if (!nome || !email || !senha || !login) {
      return {
        sucesso: false,
        erro: "Todos os campos são obrigatórios",
      };
    }

    if (senha.length < 6) {
      return {
        sucesso: false,
        erro: "Senha deve ter pelo menos 6 caracteres",
      };
    }

    const resultado = await registrarUsuario(
      nome,
      email,
      senha,
      login,
      telefone
    );

    if (!resultado.sucesso) {
      return {
        sucesso: false,
        erro: resultado.erro,
      };
    }

    // Sucesso: redirecionar para login
    setTimeout(() => {
      window.location.href = "login.html";
    }, 1500);

    return { sucesso: true };
  } catch (erro) {
    console.error("Erro ao registrar:", erro);
    return {
      sucesso: false,
      erro: "Erro ao registrar",
    };
  }
}

// ===============================================
// Fazer Logout
// ===============================================

function fazerLogout() {
  // Limpar localStorage
  localStorage.removeItem("token");
  localStorage.removeItem("usuario");

  // Limpar estado
  usuarioLogado = null;
  tokenAtual = null;

  // Atualizar UI
  atualizarUIAutenticacao();

  // Redirecionar
  window.location.href = "../index.html";
}

// ===============================================
// Atualizar UI (mostrar/esconder elementos)
// ===============================================

function atualizarUIAutenticacao() {
  const linkCadastrar = document.getElementById("link-cadastrar-servico");
  const linkPerfil = document.getElementById("link-perfil");
  const linkLogout = document.getElementById("link-logout");
  const linkLogin = document.getElementById("link-login");
  const linkRegistro = document.getElementById("link-registro");
  const nomeUsuario = document.getElementById("nome-usuario");

  if (usuarioLogado) {
    // Logado: mostrar menu autenticado
    if (linkCadastrar) linkCadastrar.style.display = "inline";
    if (linkPerfil) linkPerfil.style.display = "inline";
    if (linkLogout) linkLogout.style.display = "inline";
    if (linkLogin) linkLogin.style.display = "none";
    if (linkRegistro) linkRegistro.style.display = "none";

    if (nomeUsuario) {
      nomeUsuario.textContent = usuarioLogado.nome;
    }
  } else {
    // Não logado: mostrar menu público
    if (linkCadastrar) linkCadastrar.style.display = "none";
    if (linkPerfil) linkPerfil.style.display = "none";
    if (linkLogout) linkLogout.style.display = "none";
    if (linkLogin) linkLogin.style.display = "inline";
    if (linkRegistro) linkRegistro.style.display = "inline";
  }
}

// Inicializar ao carregar página
document.addEventListener("DOMContentLoaded", inicializarAutenticacao);
```

---

## 🟢 Passo 4: Criar `js/servicos.js` - Gerenciar Serviços

```javascript
/**
 * Gerenciar Serviços - ObraConnect
 */

// Carregar lista de serviços
async function carregarServicos() {
  try {
    const container = document.getElementById("servicos-container");
    if (!container) return;

    // Mostrar loading
    container.innerHTML =
      '<div class="text-center"><div class="spinner-border" role="status"></div></div>';

    const resultado = await listarServicos();

    if (!resultado.sucesso) {
      container.innerHTML = `<div class="alert alert-danger">Erro: ${resultado.erro}</div>`;
      return;
    }

    const servicos = resultado.dados;

    if (servicos.length === 0) {
      container.innerHTML = `<div class="alert alert-info">Nenhum serviço cadastrado.</div>`;
      return;
    }

    // Renderizar cards
    container.innerHTML = "";
    servicos.forEach((servico) => {
      const card = criarCardServico(servico);
      container.appendChild(card);
    });
  } catch (erro) {
    console.error("Erro:", erro);
  }
}

// Criar card HTML do serviço
function criarCardServico(servico) {
  const div = document.createElement("div");
  div.className = "col-12 col-sm-6 col-lg-4";

  const notaMedia = parseFloat(servico.nota_media || 0);
  const estrelas = renderizarEstrelas(notaMedia);
  const imagemUrl = servico.imagem_url || null;

  const imagemHTML = imagemUrl
    ? `<img src="${imagemUrl}" class="card-img-top" 
       alt="${servico.titulo}" 
       style="height: 200px; object-fit: cover;">`
    : `<div style="height: 200px; background: #e0e7ff; display: flex; align-items: center; justify-content: center;">
        <i class="bi bi-building" style="font-size: 3rem;"></i>
      </div>`;

  div.innerHTML = `
    <div class="card h-100">
      ${imagemHTML}
      <div class="card-body">
        <h5 class="card-title">${servico.titulo}</h5>
        <p class="card-text text-muted">${servico.desc_servico.substring(0, 100)}...</p>
        <small class="text-muted">
          <i class="bi bi-person"></i> ${servico.nome_usuario}
        </small>
      </div>
      <div class="card-footer bg-white">
        <div class="d-flex justify-content-between align-items-center mb-2">
          <div>${estrelas}</div>
          <span class="badge bg-primary">${notaMedia.toFixed(1)}</span>
        </div>
        <a href="pages/detalhes-servico.html?id=${servico.id}" 
           class="btn btn-sm btn-primary w-100">
          Ver Detalhes
        </a>
      </div>
    </div>
  `;

  return div;
}

// Renderizar estrelas de avaliação
function renderizarEstrelas(nota) {
  let html = "";
  const inteiro = Math.floor(nota);
  const decimal = nota - inteiro;

  // Estrelas cheias
  for (let i = 0; i < inteiro; i++) {
    html += '<i class="bi bi-star-fill text-warning"></i>';
  }

  // Meia estrela
  if (decimal >= 0.5) {
    html += '<i class="bi bi-star-half text-warning"></i>';
  }

  // Estrelas vazias
  const vazias = 5 - Math.ceil(nota);
  for (let i = 0; i < vazias; i++) {
    html += '<i class="bi bi-star text-warning"></i>';
  }

  return html;
}

// Carregar ao abrir página
document.addEventListener("DOMContentLoaded", carregarServicos);
```

---

## 🟢 Passo 5: Criar `index.html` - Página Principal

```html
<!DOCTYPE html>
<html lang="pt-br">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>ObraConnect - Marketplace de Serviços</title>
  
  <!-- Bootstrap CSS -->
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet" />
  
  <!-- Bootstrap Icons -->
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.0/font/bootstrap-icons.css" />
  
  <!-- Custom CSS -->
  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <!-- NAVBAR -->
  <nav class="navbar navbar-expand-lg navbar-dark bg-primary sticky-top">
    <div class="container-fluid">
      <a class="navbar-brand" href="index.html">
        <i class="bi bi-hammer"></i> ObraConnect
      </a>
      
      <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav">
        <span class="navbar-toggler-icon"></span>
      </button>
      
      <div class="collapse navbar-collapse" id="navbarNav">
        <div class="navbar-nav ms-auto">
          <a class="nav-link" href="index.html">Início</a>
          
          <!-- Links autenticado -->
          <a id="link-cadastrar-servico" class="nav-link d-none" href="pages/cadastrar-servico.html">
            Novo Serviço
          </a>
          <a id="link-perfil" class="nav-link d-none" href="pages/meu-perfil.html">
            <i class="bi bi-person"></i> Perfil
          </a>
          <a id="link-logout" class="nav-link d-none" href="#" onclick="fazerLogout()">
            Sair
          </a>
          
          <!-- Links público -->
          <a id="link-login" class="nav-link" href="pages/login.html">Login</a>
          <a id="link-registro" class="nav-link" href="pages/registro.html">Registrar</a>
        </div>
      </div>
    </div>
  </nav>
  
  <!-- JUMBOTRON -->
  <div class="bg-light py-5">
    <div class="container">
      <h1 class="display-4">Bem-vindo ao ObraConnect</h1>
      <p class="lead">Marketplace de Serviços de Construção e Reformas</p>
    </div>
  </div>
  
  <!-- LISTAGEM DE SERVIÇOS -->
  <main class="container my-5">
    <h2 class="mb-4">Serviços Disponíveis</h2>
    <div id="servicos-container" class="row g-4"></div>
  </main>
  
  <!-- FOOTER -->
  <footer class="bg-dark text-white py-4 mt-5">
    <div class="container text-center">
      <p>&copy; 2026 ObraConnect. Todos os direitos reservados.</p>
    </div>
  </footer>
  
  <!-- Scripts -->
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
  <script src="js/api.js"></script>
  <script src="js/auth.js"></script>
  <script src="js/servicos.js"></script>
</body>
</html>
```

---

## 🎨 Passo 6: Criar `style.css` - Estilos Globais

```css
:root {
  --cor-primaria: #0d6efd;
  --cor-secundaria: #6c757d;
  --cor-sucesso: #198754;
  --cor-erro: #dc3545;
  --cor-aviso: #ffc107;
}

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
  background-color: #f8f9fa;
}

/* Cards */
.card {
  border: none;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.card:hover {
  transform: translateY(-5px);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.15);
}

.card-img-top {
  height: 200px;
  object-fit: cover;
}

/* Botões */
.btn {
  border-radius: 8px;
  font-weight: 500;
  transition: all 0.3s ease;
}

.btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

/* Forms */
.form-control:focus {
  border-color: var(--cor-primaria);
  box-shadow: 0 0 0 3px rgba(13, 110, 253, 0.25);
}

/* Starling */
.star-rating {
  font-size: 1.2rem;
  color: var(--cor-aviso);
}

/* Responsive */
@media (max-width: 768px) {
  h1 {
    font-size: 2rem;
  }
  
  .container {
    padding: 0 1rem;
  }
}
```

---

## 📄 Passo 7: Criar Páginas Adicionais

**`pages/login.html`**
```html
<!-- Similar ao exemplo anterior -->
<!-- Form com login/email e senha -->
```

**`pages/registro.html`**
```html
<!-- Form com nome, email, senha, login, telefone -->
```

**`pages/cadastrar-servico.html`**
```html
<!-- Form multipart com título, descrição, categoria, imagem -->
```

**`pages/detalhes-servico.html`**
```html
<!-- Mostra serviço + avaliações -->
<!-- Permite avaliar se logado -->
```

---

## ✅ Checklist

- [ ] Pasta `frontend/` estruturada
- [ ] `js/api.js` criado com funções de requisição
- [ ] `js/auth.js` criado com autenticação
- [ ] `js/servicos.js` criado com renderização
- [ ] `index.html` criado
- [ ] `style.css` criado
- [ ] Todas páginas criadas
- [ ] Frontend testado

---

✅ **Frontend completo!** Próximo e último passo: Integração e deployment.
