# Checklist de implantação + validações e erros comuns

Este arquivo oferece um resumo rápido para rodar a aplicação e resolver problemas frequentes em backend, frontend e banco de dados.

## 1. Setup inicial

- [ ] Clonar repositório:
  - `git clone https://github.com/Vilander/projeto-obraConnect.git`
- [ ] Instalar dependências backend:
  - `cd backend`
  - `npm install`
- [ ] Criar arquivo `.env` com:
  - `DB_HOST=localhost`
  - `DB_USER=root`
  - `DB_PASSWORD=sua_senha`
  - `DB_NAME=obraconnect_db`
  - `DB_PORT=3306`
  - `JWT_SECRET=chave_secreta_forte`
  - `JWT_EXPIRY=1h`
  - `PORT=3001`
  - `NODE_ENV=development`
- [ ] Configurar banco de dados MySQL e importar `_db/obraconnect_db.sql`.
- [ ] Rodar servidor backend:
  - `npx nodemon index.js` ou `node index.js`
- [ ] Rodar frontend:
  - usar extensão Live Server do VS Code ou:
  - `npx serve frontend` (instalar serve se precisar)

## 2. Validações importantes (funcionamento correto)

### 2.1. API (backend)
- Verifique que as rotas são:
  - `POST /api/auth/registro`
  - `POST /api/auth/login`
  - `GET /api/auth/perfil`
  - `PUT /api/auth/tornar-prestador`
  - `GET /api/servicos` / `GET /api/servicos/:id` / `GET /api/servicos/meus/servicos` / `POST /api/servicos` / `PUT /api/servicos/:id` / `PATCH /api/servicos/:id/desativar` / `PATCH /api/servicos/:id/ativar` / `DELETE /api/servicos/:id`
  - `GET /api/avaliacoes/servico/:id` / `GET /api/avaliacoes/meu-historico` / `POST /api/avaliacoes` / `DELETE /api/avaliacoes/:id`

### 2.2. Frontend
- Verifique se `frontend/js/api.js`, `auth.js`, `servicos.js` estão importados em todas as páginas certas.
- Verifique IDs de elementos: `form-login`, `form-registro`, `servicos-container`, etc.
- Verifique se o token está salvo em `localStorage` (key: `token`).

## 3. Erros comuns e soluções

### 3.1. CORS
- Sintoma: `Access to fetch at ... from origin ... has been blocked by CORS policy.`
- Solução:
  - No backend (`index.js`): `app.use(cors({ origin: '*', credentials: true }));`
  - Em produção, refinar para `origin: 'https://seu-dominio.com'`.

### 3.2. Token inválido / expirado
- Sintoma: `401 Token não fornecido` ou `403 Token inválido`.
- Verifique:
  - `process.env.JWT_SECRET` e `JWT_EXPIRY` no `.env`.
  - Token no frontend (`localStorage.getItem('token')`).
  - Atualize login e reconecte.

### 3.3. Erro 500 no backend
- Sintoma: `ERR_INTERNAL_SERVER_ERROR` ou logs `Erro no servidor:`.
- Solução:
  - Confira console do backend.
  - Verifique query SQL no `routes` e `database`.
  - Habilite `NODE_ENV=development` para mensagem completa.

### 3.4. Conexão ao MySQL
- Sintoma: `ER_ACCESS_DENIED_ERROR`, `ER_BAD_DB_ERROR`, `ECONNREFUSED`.
- Solução:
  - Verifique credenciais `.env`.
  - Verifique serviço MySQL rodando.
  - Confirme database existe e schema está criado.

### 3.5. SQL injection (prevenção)
- Sintoma: consultas quebrando com `'` ou `;`.
- Solução:
  - já usa placeholders `?` em todas as queries (`banco.query(sql, [params])`), prevenindo injeção.
  - Reserve `mysql2` com query parameters.

### 3.6. Upload de imagem falha
- Sintoma: retorna erro do Multer, arquivo não salva.
- Verifique:
  - `uploadDir` existe e app tem permissão.
  - `Content-Type` não deve ser `application/json` para `formData`.
  - Tamanho máximo configurado: `MAX_FILE_SIZE` no `.env` ou 5MB padrão.

### 3.7. Rota 404 no backend
- Sintoma: `Rota não encontrada.`
- Verifique URL e método. Ex: `/api/servicos` vs `/servicos` no frontend.

## 4. Teste de ponta a ponta

1. Registrar usuário.
2. Fazer login e conferir token no storage.
3. Acessar `meu-perfil.html` (deve carregar info do usuário).
4. Tornar prestador e validar botão de novo serviço.
5. Cadastrar serviço com imagem.
6. Acessar serviço em detalhes; verificar informações do prestador.
7. Fazer avaliação de outro usuário (crie segundo usuário).
8. Listar e deletar avaliação.
9. Deletar serviço e confirmar no banco.

## 5. Melhores práticas para deploy

- Backup do banco antes de rodar script.
- Use variáveis ambiente seguras fora do `.env` em produção (Azure, AWS, Docker secrets).
- Regras CORS estritas em produção.
- Não comite `.env` ao Git.
- Use HTTPS.
- Monitore 5xx e 4xx.

---

> Você agora tem três arquivos de estudo no `ESTUDOS/PASSO_A_PASSO`:
> - `00-REPLICAR-PROJETO-DO-ZERO.md`
> - `01-CODIGO-ATUAL-COM-COMENTARIOS.md`
> - `02-FRONTEND-CODIGO-ATUAL-COM-COMENTARIOS.md`
> - `03-CHECKLIST-VALIDACOES-ERROS.md`

Bom trabalho: com isso a documentação ficou completa e prática.  