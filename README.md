# ObraConnect

Marketplace para conectar profissionais de construção e reformas com clientes.

**Stack**: Node.js + Express + MySQL | HTML5 + Bootstrap + JavaScript

---

## Funcionalidades

- **Autenticação**: Registro, login com JWT e tipos de usuário (cliente, prestador, admin)
- **Serviços**: Criar, editar, deletar serviços com upload de imagens e categorias
- **Avaliações**: Sistema de 4 notas (preço, tempo, higiene, educação) + comentários
- **Perfil**: Visualizar e gerenciar dados pessoais e serviços

---

## Requisitos

- Node.js 14+
- MySQL 5.7+
- Navegador moderno

---

## Instalação

### 1. Banco de Dados

```bash
mysql -u root -p obraconnect_db < _db/obraconnect_db.sql
```

Ou via phpMyAdmin: abra http://localhost/phpmyadmin, importe o arquivo `_db/obraconnect_db.sql`

### 2. Backend

```bash
cd backend
npm install
```

3. Crie arquivo `.env`:

```
PORT=3001
NODE_ENV=development
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=
DB_NAME=obraconnect_db
JWT_SECRET=sua_chave_secreta_super_segura_aqui_123456789
```

**Se MySQL tem senha**: adicione em `DB_PASSWORD=sua_senha`

4. Inicie o servidor:

```bash
npm run dev
```

Deve mostrar: "Servidor rodando em: http://localhost:3001"

### 3. Frontend

Abra `frontend/index.html` no navegador ou use uma extensão como Live Server.

---

## Como Usar

**Registrar**: Na página inicial, clique em "Registrar" e preencha os dados. Você inicia como "usuário".

**Virar Prestador**: Faça login e clique em "Virar Prestador" no seu perfil.

**Anunciar Serviço**: Clique em "Novo Serviço", preencha Título, Descrição, Categoria e foto. O serviço aparece na homepage.

**Avaliar Serviço**: Abra um serviço, preencha as 4 notas (preço, tempo, higiene, educação) e comentário.

---

## Estrutura

```
projeto-obraConnect/
├── backend/
│   ├── config/          (database.js, upload.js)
│   ├── middlewares/     (autenticacao.js)
│   ├── routes/          (authRoutes.js, servicoRoutes.js, avaliacaoRoutes.js)
│   ├── uploads/         (imagens dos serviços)
│   ├── index.js
│   └── package.json
├── frontend/
│   ├── index.html       (home/listagem)
│   ├── style.css
│   ├── pages/           (login, registro, cadastrar-servico, detalhes-servico, meu-perfil)
│   └── js/              (api.js, auth.js, servicos.js)
├── _db/
│   └── obraconnect_db.sql
└── README.md
```

---

## Usuários de Teste

| Usuário | Email                 | Senha    | Tipo      |
| ------- | --------------------- | -------- | --------- |
| admin   | vilander...@gmail.com | admin123 | admin     |
| jose    | jose@email.com        | senha123 | prestador |
| aline   | aline@teste.com       | senha123 | usuario   |

---

## Segurança e Criptografia

**Bcrypt** - Criptografia de Senhas

- Algoritmo: bcrypt com salt rounds = 10
- Função: `bcryptjs` criptografa senhas antes de salvar no banco
- Verificação: Ao fazer login, a senha inserida é comparada com o hash armazenado
- Vantagem: Impede que senhas sejam recuperadas mesmo com acesso ao banco de dados

**JWT** - Autenticação de Sessão

- Algoritmo: HS256 (HMAC SHA-256)
- Token gerado com: ID do usuário, nome, email, tipo de usuário
- Validade: 7 dias (configurável)
- Armazenamento: Salvo no `localStorage` do navegador
- Uso: Enviado em cada requisição no header `Authorization: Bearer TOKEN`
- Benefício: Autenticação stateless, sem necessidade de sessão no servidor

---

## Problemas Comuns

**Erro ao conectar banco**: Verificar se MySQL está rodando, se `.env` está correto, criar banco `obraconnect_db`.

**Imagens não aparecem**: Verificar se pasta `backend/uploads/` existe e se o arquivo foi salvo.

**Botão "Novo Serviço" não aparece**: Fazer login e clicar "Virar Prestador".

---

## Licença

MIT License - Use livremente!

---

## Autor

Vilander Costa

Desenvolvido como um projeto de aprendizado.
