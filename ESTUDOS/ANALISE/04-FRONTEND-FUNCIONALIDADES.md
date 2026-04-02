# 🌐 ANÁLISE - FRONTEND E INTEGRAÇÕES

## 📋 Visão Geral

O frontend do ObraConnect é uma **Single Page Application** construída com **HTML5 + CSS3 + JavaScript Vanilla** (sem frameworks como React).

### Estrutura

```
frontend/
├── index.html              # 🟢 Página principal
├── style.css               # 🟢 Estilos globais
├── assets/
│   └── imgs/              # Imagens do projeto
├── js/
│   ├── api.js             # 🟢 Funções de requisição
│   ├── auth.js            # 🟢 Autenticação e login
│   └── servicos.js        # 🟢 Gerenciamento de serviços
└── pages/
    ├── login.html         # 🟢 Tela de login
    ├── registro.html      # 🟢 Tela de registro
    ├── meu-perfil.html    # 🟢 Perfil do usuário
    ├── cadastrar-servico.html  # Criar serviço
    └── detalhes-servico.html   # Ver detalhes
```

---

## 🟢 Arquivo: `js/api.js` - Camada de Comunicação

**Responsabilidade**: Abstrair requisições HTTP e upload de forma reutilizável

### 1. Função genérica `fazerRequisicao()`

```javascript
async function fazerRequisicao(endpoint, metodo = "GET", dados = null) {
  try {
    const opcoes = {
      method: metodo,
      headers: { "Content-Type": "application/json" }
    };
    
    // Adicionar token se existir (autenticação)
    const token = localStorage.getItem("token");
    if (token) {
      opcoes.headers["Authorization"] = `Bearer ${token}`;
    }
    
    // Adicionar corpo se existir
    if (dados) {
      opcoes.body = JSON.stringify(dados);
    }
    
    const resposta = await fetch(`${API_BASE_URL}${endpoint}`, opcoes);
    const resultado = await resposta.json();
    
    if (!resposta.ok) {
      throw new Error(resultado.erro || "Erro na requisição");
    }
    
    return { sucesso: true, dados: resultado };
  } catch (erro) {
    return { sucesso: false, erro: erro.message };
  }
}
```

**Por que abstrair?**

```
❌ Sem abstração: Repetir código em cada requisição
POST /api/auth/login
POST /api/auth/registro
GET /api/servicos
PUT /api/servicos/:id
... (repetir headers, token, error handling em cada um)

✅ Com abstração: Reutilizar função genérica
fazerRequisicao("/auth/login", "POST", dados)
fazerRequisicao("/auth/registro", "POST", dados)
fazerRequisicao("/servicos", "GET")
fazerRequisicao("/servicos/1", "PUT", dados)
... (tudo passa por verificações centralizadas)
```

### 2. Função `fazerUpload()`

```javascript
async function fazerUpload(endpoint, arquivo) {
  try {
    const formData = new FormData();
    formData.append("imagem", arquivo);
    
    const opcoes = { method: "POST" };
    
    const token = localStorage.getItem("token");
    if (token) {
      opcoes.headers = { Authorization: `Bearer ${token}` };
    }
    
    opcoes.body = formData;  // FormData, não JSON
    
    const resposta = await fetch(`${API_BASE_URL}${endpoint}`, opcoes);
    // ... resto do código
  }
}
```

**Por que FormData?**

```
❌ JSON não suporta arquivos
JSON.stringify({imagem: File})  // ❌ Não funciona

✅ FormData é feito para arquivos
const fd = new FormData();
fd.append("imagem", arquivo);   // ✅ Funciona!
```

### 3. Funções de Autenticação

```javascript
// Registro
async function registrarUsuario(nome, email, senha, login, telefone) {
  return fazerRequisicao("/auth/registro", "POST", {
    nome_usuario: nome,
    email, senha, login, telefone
  });
}

// Login
async function fazerLogin(login, senha) {
  return fazerRequisicao("/auth/login", "POST", { login, senha });
}

// Ver perfil
async function verPerfil() {
  return fazerRequisicao("/auth/perfil", "GET");
}

// Tornar prestador
async function tornarPrestador() {
  return fazerRequisicao("/auth/tornar-prestador", "PUT");
}
```

### 4. Funções de Serviços

```javascript
// Listar serviços
async function listarServicos() {
  return fazerRequisicao("/servicos", "GET");
}

// Ver detalhes
async function buscarServicoPorId(id) {
  return fazerRequisicao(`/servicos/${id}`, "GET");
}

// Meus serviços
async function listarMeusServicos() {
  return fazerRequisicao("/servicos/meus/servicos", "GET");
}

// Criar com imagem
async function criarServico(titulo, descricao, id_categoria, arquivo) {
  const formData = new FormData();
  formData.append("titulo", titulo);
  formData.append("descricao", descricao);
  formData.append("id_categoria", id_categoria);
  if (arquivo) formData.append("imagem", arquivo);
  
  // Requisição com FormData
  return fazerUpload("/servicos", formData);
}

// Editar serviço
async function atualizarServico(id, titulo, descricao) {
  return fazerRequisicao(`/servicos/${id}`, "PUT", {
    titulo, descricao
  });
}

// Deletar
async function deletarServico(id) {
  return fazerRequisicao(`/servicos/${id}`, "DELETE");
}
```

### 5. Funções de Avaliações

```javascript
// Listar avaliações de um serviço
async function listarAvaliacoes(id_servico) {
  return fazerRequisicao(`/avaliacoes/servico/${id_servico}`, "GET");
}

// Minhas avaliações
async function listarMinhasAvaliacoes() {
  return fazerRequisicao("/avaliacoes/meu-historico", "GET");
}

// Criar avaliação
async function criarAvaliacao(id_servico, nota_preco, nota_tempo, nota_higiene, nota_educacao, comentario) {
  return fazerRequisicao("/avaliacoes", "POST", {
    id_servico,
    nota_preco,
    nota_tempo_execucao: nota_tempo,
    nota_higiene,
    nota_educacao,
    comentario
  });
}

// Deletar avaliação
async function deletarAvaliacao(id) {
  return fazerRequisicao(`/avaliacoes/${id}`, "DELETE");
}
```

---

## 🟢 Arquivo: `js/auth.js` - Gerenciamento de Autenticação

**Responsabilidade**: Centralizar lógica de login, logout e estado do usuário

### 1. Estado Global

```javascript
let usuarioLogado = null;    // Dados do usuário logado em memória
let tokenAtual = null;       // JWT token em memória

// Persistência:
localStorage.token           // Token salvo no navegador
localStorage.usuario         // Dados do usuário salvos em JSON
```

**Por que localStorage?**

```
estado em memória (RAM):
- Rápido de acessar
- Perdido ao recarregar página

localStorage (disco):
- Persiste entre sessões
- Permanece após fechar navegador
- Até limpar cache ou logout

Fluxo:
1. User faz login
2. Backend retorna token + usuário
3. JavaScript salva em localStorage
4. User recarrega página
5. JavaScript lê localStorage
6. Continua logado sem novo login
```

### 2. Inicializar Autenticação

```javascript
function inicializarAutenticacao() {
  const token = localStorage.getItem("token");
  const usuario = localStorage.getItem("usuario");
  
  if (token && usuario) {
    try {
      usuarioLogado = JSON.parse(usuario);
      tokenAtual = token;
      atualizarUIAutenticacao();  // Mostrar/esconder elementos
    } catch (erro) {
      fazerLogout();  // Dados corrompidos
    }
  }
}

// Chamar ao carregar página
document.addEventListener("DOMContentLoaded", inicializarAutenticacao);
```

### 3. Realizar Login

```javascript
async function realizarLogin(loginOuEmail, senha) {
  try {
    const resultado = await fazerLogin(loginOuEmail, senha);
    
    if (!resultado.sucesso) {
      return { sucesso: false, erro: resultado.erro };
    }
    
    // Guardar dados
    const { token, usuario } = resultado.dados;
    localStorage.setItem("token", token);
    localStorage.setItem("usuario", JSON.stringify(usuario));
    
    // Atualizar memória
    usuarioLogado = usuario;
    tokenAtual = token;
    
    // Atualizar UI
    atualizarUIAutenticacao();
    
    // Redirecionar
    window.location.href = "index.html";
    
    return { sucesso: true, usuario };
  } catch (erro) {
    return { sucesso: false, erro: "Erro ao fazer login" };
  }
}
```

### 4. Realizar Registro

```javascript
async function realizarRegistro(nome, email, senha, login, telefone) {
  try {
    // Validações no front
    if (!nome || !email || !senha || !login) {
      return { sucesso: false, erro: "Campos obrigatórios" };
    }
    
    if (senha.length < 6) {
      return { sucesso: false, erro: "Senha mín. 6 caracteres" };
    }
    
    const resultado = await registrarUsuario(nome, email, senha, login, telefone);
    
    if (!resultado.sucesso) {
      return { sucesso: false, erro: resultado.erro };
    }
    
    // Sucesso: redirecionar para login
    window.location.href = "login.html";
    return { sucesso: true };
  } catch (erro) {
    return { sucesso: false, erro: "Erro ao registrar" };
  }
}
```

### 5. Fazer Logout

```javascript
function fazerLogout() {
  // Limpar localStorage
  localStorage.removeItem("token");
  localStorage.removeItem("usuario");
  
  // Limpar memória
  usuarioLogado = null;
  tokenAtual = null;
  
  // Atualizar UI
  atualizarUIAutenticacao();
  
  // Redirecionar
  window.location.href = "index.html";
}
```

### 6. Atualizar UI Baseado em Autenticação

```javascript
function atualizarUIAutenticacao() {
  const linkCadastrar = document.getElementById("link-cadastrar-servico");
  const linkPerfil = document.getElementById("link-perfil");
  const linkLogout = document.getElementById("link-logout");
  const linkLogin = document.getElementById("link-login");
  const linkRegistro = document.getElementById("link-registro");
  const nomeUsuario = document.getElementById("nome-usuario");
  
  if (usuarioLogado) {
    // User logado: mostrar menu autenticado
    linkCadastrar?.style.display = "inline";
    linkPerfil?.style.display = "inline";
    linkLogout?.style.display = "inline";
    linkLogin?.style.display = "none";
    linkRegistro?.style.display = "none";
    
    if (nomeUsuario) {
      nomeUsuario.textContent = usuarioLogado.nome;
    }
  } else {
    // User não logado: mostrar menu público
    linkCadastrar?.style.display = "none";
    linkPerfil?.style.display = "none";
    linkLogout?.style.display = "none";
    linkLogin?.style.display = "inline";
    linkRegistro?.style.display = "inline";
  }
}
```

---

## 🟢 Arquivo: `js/servicos.js` - Gerenciamento de Serviços

**Responsabilidade**: Renderizar, criar, editar, deletar serviços

### 1. Carregar Serviços (index.html)

```javascript
async function carregarServicos() {
  try {
    const container = document.getElementById("servicos-container");
    if (!container) return;
    
    // Mostrar loading
    container.innerHTML = '<div class="spinner-border">Carregando...</div>';
    
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
    
    // Renderizar cada serviço
    container.innerHTML = "";
    servicos.forEach((servico) => {
      const card = criarCardServico(servico);
      container.appendChild(card);
    });
  } catch (erro) {
    console.error("Erro:", erro);
  }
}

// Chamar ao carregar página
document.addEventListener("DOMContentLoaded", carregarServicos);
```

### 2. Criar Card de Serviço

```javascript
function criarCardServico(servico) {
  const div = document.createElement("div");
  div.className = "col-12 col-sm-6 col-lg-4";
  
  const notaMedia = parseFloat(servico.nota_media || 0);
  const estrelas = renderizarEstrelas(notaMedia);
  
  const imagemUrl = servico.imagem_url || null;
  const imagemHTML = imagemUrl 
    ? `<img src="${imagemUrl}" alt="${servico.titulo}" onerror="this.style.display='none';">`
    : `<div class="card-img-fallback"><i class="bi bi-building"></i></div>`;
  
  div.innerHTML = `
    <div class="card h-100">
      <div style="background-color: var(--azul-claro); height: 250px;">
        ${imagemHTML}
      </div>
      <div class="card-body">
        <h5 class="card-title">${servico.titulo}</h5>
        <p class="card-text">${servico.desc_servico}</p>
        <small><i class="bi bi-person"></i> ${servico.nome_usuario}</small>
      </div>
      <div class="card-footer">
        <div class="rating-display">
          <div class="stars">${estrelas}</div>
          <span>${notaMedia.toFixed(1)}</span>
        </div>
        <a href="pages/detalhes-servico.html?id=${servico.id}" class="btn btn-sm btn-primary">
          Ver Detalhes
        </a>
      </div>
    </div>
  `;
  
  return div;
}
```

### 3. Renderizar Estrelas

```javascript
function renderizarEstrelas(nota) {
  // Nota 4.75 → 4 estrelas cheias + 1 meia
  const inteiro = Math.floor(nota);
  const decimal = nota - inteiro;
  
  let html = "";
  
  // Estrelas cheias
  for (let i = 0; i < inteiro; i++) {
    html += '<i class="bi bi-star-fill"></i>';
  }
  
  // Meia estrela se decimal >= 0.5
  if (decimal >= 0.5) {
    html += '<i class="bi bi-star-half"></i>';
  }
  
  // Estrelas vazias
  const vazias = 5 - Math.ceil(nota);
  for (let i = 0; i < vazias; i++) {
    html += '<i class="bi bi-star"></i>';
  }
  
  return html;
}
```

### 4. Ver Detalhes do Serviço

```javascript
async function carregarDetalhesServico(id) {
  try {
    // Buscar serviço
    const resultado = await buscarServicoPorId(id);
    
    if (!resultado.sucesso) {
      mostrarAviso(`Erro: ${resultado.erro}`, "danger");
      return;
    }
    
    const servico = resultado.dados;
    
    // Preencher HTML
    document.getElementById("servico-titulo").textContent = servico.titulo;
    document.getElementById("servico-prestador").textContent = servico.nome_usuario;
    document.getElementById("servico-descricao").textContent = servico.desc_servico;
    document.getElementById("servico-nota").textContent = servico.nota_media.toFixed(1);
    document.getElementById("servico-avaliacoes").textContent = servico.total_avaliacoes;
    
    // Carregar avaliações
    await carregarAvaliacoes(id);
  } catch (erro) {
    console.error("Erro:", erro);
  }
}
```

### 5. Criar Novo Serviço

```javascript
async function criarNovoServico(e) {
  e.preventDefault();
  
  const titulo = document.getElementById("titulo").value;
  const descricao = document.getElementById("descricao").value;
  const id_categoria = document.getElementById("categoria").value;
  const arquivo = document.getElementById("imagem").files[0];
  
  // Validar
  if (!titulo || !descricao) {
    mostrarAviso("Preencha todos os campos!", "danger");
    return;
  }
  
  // Desabilitar botão
  document.getElementById("btn-criar").disabled = true;
  
  const resultado = await criarServico(titulo, descricao, id_categoria, arquivo);
  
  if (!resultado.sucesso) {
    mostrarAviso(`Erro: ${resultado.erro}`, "danger");
    document.getElementById("btn-criar").disabled = false;
    return;
  }
  
  mostrarAviso("Serviço criado com sucesso!", "success");
  setTimeout(() => window.location.href = "index.html", 1500);
}

document.getElementById("form-servico")?.addEventListener("submit", criarNovoServico);
```

### 6. Editar Serviço

```javascript
async function editarServico(id_servico) {
  const novoTitulo = prompt("Novo título:");
  if (!novoTitulo) return;
  
  const resultado = await atualizarServico(id_servico, novoTitulo, descricaoAtual);
  
  if (resultado.sucesso) {
    mostrarAviso("Serviço atualizado!", "success");
    location.reload();
  } else {
    mostrarAviso(`Erro: ${resultado.erro}`, "danger");
  }
}
```

### 7. Deletar Serviço

```javascript
async function deletarServico(id_servico) {
  if (!confirm("Tem certeza que deseja deletar?")) return;
  
  const resultado = await deletarServico(id_servico);
  
  if (resultado.sucesso) {
    mostrarAviso("Serviço deletado!", "success");
    window.location.href = "index.html";
  } else {
    mostrarAviso(`Erro: ${resultado.erro}`, "danger");
  }
}
```

---

## 🎨 Arquivo: `style.css` - Estilos

**Responsabilidade**: Estilizar toda a aplicação com Bootstrap

### Variáveis CSS Customizadas

```css
:root {
  --azul-marinho: #1e3a8a;      /* Cor primária */
  --azul-claro: #dbeafe;         /* Fundo claro */
  --verde-sucesso: #10b981;       /* Botões sucesso */
  --vermelho-erro: #ef4444;       /* Avisos erro */
  --cinza-texto: #6b7280;         /* Texto secundário */
}
```

### Layouts Responsivos

```css
/* Mobile First */
.servicos-grid {
  display: grid;
  grid-template-columns: 1fr;  /* 1 coluna em mobile */
}

/* Tablet */
@media (min-width: 768px) {
  .servicos-grid {
    grid-template-columns: repeat(2, 1fr);  /* 2 colunas */
  }
}

/* Desktop */
@media (min-width: 1024px) {
  .servicos-grid {
    grid-template-columns: repeat(3, 1fr);  /* 3 colunas */
  }
}
```

---

## 📄 Arquivos HTML - Estrutura

### `index.html` - Página Principal

```html
<!DOCTYPE html>
<html lang="pt-br">
<head>
  <meta charset="UTF-8">
  <title>ObraConnect - Marketplace</title>
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <!-- Navbar -->
  <nav class="navbar">
    <a href="index.html">🏗️ ObraConnect</a>
    <div id="menu-autenticado">
      <a href="pages/cadastrar-servico.html">Novo Serviço</a>
      <a href="pages/meu-perfil.html">Perfil</a>
      <a onclick="fazerLogout()">Logout</a>
    </div>
    <div id="menu-publico">
      <a href="pages/login.html">Login</a>
      <a href="pages/registro.html">Registro</a>
    </div>
  </nav>
  
  <!-- Conteúdo -->
  <main>
    <h1>Serviços Disponíveis</h1>
    <div id="servicos-container"></div>
  </main>
  
  <!-- Scripts -->
  <script src="js/api.js"></script>
  <script src="js/auth.js"></script>
  <script src="js/servicos.js"></script>
</body>
</html>
```

### `pages/login.html` - Tela de Login

```html
<div class="form-container">
  <form id="form-login">
    <h2>Login</h2>
    
    <input type="text" id="login" placeholder="Login ou Email" required>
    <input type="password" id="senha" placeholder="Senha" required>
    
    <button type="submit" class="btn btn-primary">Entrar</button>
    <p>Não tem conta? <a href="registro.html">Registre-se</a></p>
  </form>
</div>

<script>
document.getElementById("form-login").addEventListener("submit", async (e) => {
  e.preventDefault();
  
  const login = document.getElementById("login").value;
  const senha = document.getElementById("senha").value;
  
  const resultado = await realizarLogin(login, senha);
  
  if (resultado.sucesso) {
    window.location.href = "../index.html";
  } else {
    alert("Erro: " + resultado.erro);
  }
});
</script>
```

---

## 🔄 Fluxo Completo - Exemplo: Criar Serviço

```
1. User clica em "Novo Serviço" (navbar)
   └─> Redireciona para pages/cadastrar-servico.html

2. Form carrega
   ├─> JavaScript valida: usuário logado?
   │   └─> Se não, redireciona para login.html
   └─> Inicializa autenticação

3. User preenche form
   ├─> Título: "Encanamento"
   ├─> Descrição: "Conserto de vazamentos"
   ├─> Categoria: "Bombeiro Hidráulico"
   └─> Imagem: seleciona arquivo.jpg

4. User clica em "Criar"
   └─> JavaScript execute criarNovoServico()
       ├─> Validar campos (frontend)
       ├─> Montar FormData com arquivo
       └─> Chamar api.js: criarServico()

5. Frontend envia requisição para Backend
   └─> POST /api/servicos
       ├─ Headers: Authorization: Bearer TOKEN
       ├─ Body: FormData (arquivo + dados)
       └─ Multer recebe e processa

6. Backend processa
   ├─> Verificar token (middleware)
   ├─> Verificar if prestador (middleware)
   ├─> Multer valida arquivo (imagem?)
   ├─> Multer salva em /uploads
   ├─> Gera URL: http://localhost:3001/uploads/imagem-timestamp.jpg
   └─> INSERT no banco com URL

7. Backend retorna sucesso
   └─> 201 Created: { id_servico: 5, imagem: "http://..." }

8. Frontend recebe resposta
   └─> Mostrar mensagem "Serviço criado!"
       └─> Redirecionar para index.html após 1.5s

9. User vê serviço novo na lista
   └─> Serviço aparece no grid com imagem e dados
```

---

## 🎯 Resumo Frontend

| Arquivo | Responsabilidade |
|---------|-----------------|
| `api.js` | Abstrair requisições HTTP |
| `auth.js` | Autenticação e estado do usuário |
| `servicos.js` | Renderizar e gerenciar serviços |
| `style.css` | Estilos responsivos com Bootstrap |
| `index.html` | Página principal com navbar |
| `pages/login.html` | Tela de login |
| `pages/registro.html` | Tela de registro |
| `pages/cadastrar-servico.html` | Criar novo serviço |
| `pages/detalhes-servico.html` | Ver detalhes e avaliações |
| `pages/meu-perfil.html` | Perfil do usuário |

---

Este é o frontend do ObraConnect! Interface moderna, responsiva e fácil de usar!
