# Explicação Simples das Funcionalidades do Projeto ObraConnect

Olá! Vou explicar de forma bem simples como funciona este projeto chamado ObraConnect. Imagine que você é um leigo total em programação e quer entender o básico. Vou dividir em três partes principais: **Front-end** (a parte que você vê e interage), **Back-end** (a parte que trabalha nos bastidores) e **Banco de Dados** (onde ficam guardados os dados). Também vou destacar pontos específicos como login, senhas e outras coisas que você mencionou.

## 1. O que é o ObraConnect?
Este é um sistema tipo "marketplace" para serviços de construção e reformas. É como um site onde pessoas podem se cadastrar como clientes ou prestadores de serviço (como pedreiros, eletricistas, etc.), e os clientes podem contratar esses profissionais.

## 2. Front-end: A Parte Visível (Interface do Usuário)
O front-end é tudo o que você vê no navegador: as páginas, botões, formulários. É feito com HTML (estrutura das páginas), CSS (aparência bonita) e JavaScript (para fazer as coisas funcionarem quando você clica).

### Como funciona:
- **Páginas principais**: Há páginas como login, registro, perfil do usuário, cadastro de serviços, etc.
- **Interação**: Quando você preenche um formulário de login e clica em "entrar", o JavaScript pega esses dados e envia para o back-end.
- **Consumo de dados**: O front-end "consome" dados do back-end através de requisições (usando uma coisa chamada "fetch" em JavaScript). Por exemplo, para mostrar uma lista de serviços, ele pede ao back-end: "me dá a lista de serviços" e o back-end responde com os dados.
- **Sessões**: Depois de logar, o front-end guarda um "token" (uma chave especial) no navegador (localStorage). Esse token é usado para provar que você está logado em outras requisições.

### Destaque: Criação de Login
- Você vai para a página de login (login.html).
- Preenche seu login/email e senha.
- O JavaScript (no arquivo auth.js) pega esses dados e envia para o back-end via uma requisição HTTP (POST para /api/auth/login).
- Se der certo, o back-end manda de volta um token JWT, que o front-end guarda.

## 3. Back-end: A Parte dos Bastidores (Servidor)
O back-end é como o "cérebro" do sistema. Ele roda em um servidor (usando Node.js e Express) e processa as requisições que vêm do front-end. Ele decide o que fazer com os dados, conversa com o banco de dados, etc.

### Como funciona:
- **Servidor**: Usa Express.js para criar um servidor web que "ouve" nas portas (normalmente 3001).
- **Rotas**: São caminhos específicos, como /api/auth/login. Cada rota faz uma coisa diferente (login, registro, etc.).
- **Acesso às rotas**: Algumas rotas são "protegidas" – só quem tem um token válido pode acessar. Há um "middleware" (autenticacao.js) que verifica o token antes de deixar passar.
- **Processamento**: Quando você faz login, o back-end pega sua senha, compara com a versão criptografada no banco, e se estiver certa, gera um token JWT.

### Destaques Específicos:
- **Criptografia da senha**: Usa uma biblioteca chamada "bcryptjs". Quando você se registra, a senha é "hasheada" (transformada em um código confuso) antes de ser salva no banco. Isso impede que alguém veja sua senha real se o banco for hackeado.
- **Uso do mysql2**: É uma biblioteca para conectar o back-end ao banco MySQL. Permite fazer consultas como "SELECT * FROM usuarios WHERE email = ?" de forma segura.
- **Utilização do JWT (JSON Web Token)**: É como um "passaporte" temporário. Quando você loga, o back-end cria um token com suas informações (id, nome, etc.) e uma chave secreta. Esse token expira depois de um tempo (por exemplo, 1 hora). O front-end manda esse token em cada requisição para provar quem é.
- **Criação de sessões**: Não usa sessões tradicionais (como cookies), mas sim JWT. O token é guardado no front-end e enviado sempre que precisa acessar algo protegido.

## 4. Banco de Dados: Onde Tudo é Guardado
O banco de dados é como uma "caixa de ferramentas organizada". Usa MySQL, que é um tipo de banco relacional. Armazena tabelas com dados como usuários, serviços, avaliações, etc.

### Como funciona:
- **Tabelas principais**:
  - `oc__tb_usuario`: Guarda dados dos usuários (nome, email, senha criptografada, tipo: cliente ou prestador).
  - `oc__tb_servico`: Serviços oferecidos (descrição, preço, categoria).
  - `oc__tb_avaliacao`: Avaliações dos serviços (notas, comentários).
  - `oc__tb_categoria`: Categorias dos serviços (pedreiro, eletricista, etc.).
- **Conexão**: O back-end usa mysql2 para conectar e fazer consultas (queries) como inserir um novo usuário ou buscar serviços.
- **Consumo dos dados**: O back-end faz queries no banco e manda os resultados para o front-end, que mostra na tela.

## 5. Como Tudo Funciona Junto (Fluxo Básico)
1. **Registro**: Você preenche o formulário no front-end. O back-end criptografa a senha com bcrypt e salva no banco MySQL.
2. **Login**: Você manda login/senha. O back-end compara a senha (usando bcrypt) e gera um JWT se estiver certa.
3. **Acesso**: Com o token, você pode acessar rotas protegidas. O middleware verifica o token.
4. **Dados**: Para ver serviços, o front-end pede ao back-end, que consulta o banco e responde.

## 6. Tecnologias Usadas (Simplificado)
- **Front-end**: HTML, CSS, JavaScript (com fetch para API).
- **Back-end**: Node.js, Express.js, bcryptjs (senhas), jsonwebtoken (JWT), mysql2 (banco).
- **Banco**: MySQL.
- **Outros**: CORS (para permitir requisições do front-end), multer (para uploads de imagens).

Espero que isso tenha ajudado a entender o básico! Se quiser mais detalhes sobre alguma parte, é só perguntar.</content>
<parameter name="filePath">c:\Users\Vilander\Documents\GitHub\projeto-obraConnect\ESTUDOS\EXPLICACAO_SIMPLES\explicacao-funcionalidades.md