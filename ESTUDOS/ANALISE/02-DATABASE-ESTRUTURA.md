# 🗄️ ANÁLISE - ESTRUTURA DO BANCO DE DADOS

## 📋 Visão Geral

O banco de dados **obraconnect_db** é um banco **relacional** que armazena dados de usuários, serviços oferecidos e avaliações entre prestadores e clientes.

- **SGBD**: MySQL 10.4.32 / MariaDB
- **Collation**: utf8mb4_general_ci (suporta emojis e caracteres especiais)
- **Padrão**: 3 tabelas principais com relacionamentos

---

## 📊 Tabelas e Relacionamentos

### 1️⃣ Tabela `oc__tb_usuario`

**Propósito**: Armazenar dados de todos os usuários (clientes e prestadores)

```sql
CREATE TABLE oc__tb_usuario (
  id INT PRIMARY KEY AUTO_INCREMENT,
  nome_usuario VARCHAR(100) NOT NULL,
  email VARCHAR(100) UNIQUE NOT NULL,
  senha VARCHAR(255) NOT NULL,
  login VARCHAR(50) UNIQUE NOT NULL,
  telefone VARCHAR(20),
  tipo_usuario ENUM('usuario', 'prestador', 'admin'),
  data_cadastro TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | INT | Identificador único, autoincremento |
| `nome_usuario` | VARCHAR(100) | Nome completo do usuário |
| `email` | VARCHAR(100) | Email único para recuperação de senha |
| `senha` | VARCHAR(255) | Senha criptografada com bcryptjs |
| `login` | VARCHAR(50) | Username único para login |
| `telefone` | VARCHAR(20) | Telefone para contato (opcional) |
| `tipo_usuario` | ENUM | 'usuario' (cliente), 'prestador' (profissional), 'admin' |
| `data_cadastro` | TIMESTAMP | Data/hora de criação automática |

**Índices**:
- `id` (PRIMARY KEY) - Busca rápida por ID
- `email` (UNIQUE) - Previne emails duplicados
- `login` (UNIQUE) - Previne logins duplicados

**Exemplos de dados**:
```json
{
  "id": 1,
  "nome_usuario": "João Silva",
  "email": "joao@email.com",
  "senha": "$2a$10$...", // bcryptjs hash
  "login": "joao_silva",
  "telefone": "11999999999",
  "tipo_usuario": "prestador",
  "data_cadastro": "2026-01-22 19:08:35"
}
```

---

### 2️⃣ Tabela `oc__tb_servico`

**Propósito**: Armazenar serviços oferecidos pelos prestadores

```sql
CREATE TABLE oc__tb_servico (
  id INT PRIMARY KEY AUTO_INCREMENT,
  id_usuario INT NOT NULL,
  titulo VARCHAR(150) NOT NULL,
  desc_servico TEXT,
  id_categoria INT,
  imagem_url VARCHAR(255),
  data_cadastro TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (id_usuario) REFERENCES oc__tb_usuario(id),
  FOREIGN KEY (id_categoria) REFERENCES oc__tb_categoria(id)
)
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | INT | Identificador único |
| `id_usuario` | INT | FK para quem oferece o serviço |
| `titulo` | VARCHAR(150) | Nome/título do serviço |
| `desc_servico` | TEXT | Descrição detalhada |
| `id_categoria` | INT | FK para categoria do serviço |
| `imagem_url` | VARCHAR(255) | URL da imagem (salva em `/uploads`) |
| `data_cadastro` | TIMESTAMP | Data de criação do serviço |

**Relacionamentos**:
- `id_usuario` → `oc__tb_usuario(id)` - O prestador que oferece
- `id_categoria` → `oc__tb_categoria(id)` - Tipo de serviço

**Exemplos de dados**:
```json
{
  "id": 1,
  "id_usuario": 1,
  "titulo": "Encanamento em Geral",
  "desc_servico": "Conserto de vazamentos, troca de canos...",
  "id_categoria": 4,
  "imagem_url": "http://localhost:3001/uploads/imagem-1234567890.jpg",
  "data_cadastro": "2026-01-25 14:18:21"
}
```

---

### 3️⃣ Tabela `oc__tb_avaliacao`

**Propósito**: Armazenar avaliações/reviews de serviços

```sql
CREATE TABLE oc__tb_avaliacao (
  id INT PRIMARY KEY AUTO_INCREMENT,
  id_servico INT NOT NULL,
  id_usuario INT NOT NULL,
  nota_preco TINYINT NOT NULL CHECK (nota_preco BETWEEN 1 AND 5),
  nota_tempo_execucao TINYINT NOT NULL CHECK (nota_tempo_execucao BETWEEN 1 AND 5),
  nota_higiene TINYINT NOT NULL CHECK (nota_higiene BETWEEN 1 AND 5),
  nota_educacao TINYINT NOT NULL CHECK (nota_educacao BETWEEN 1 AND 5),
  comentario TEXT,
  data_avaliacao TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (id_servico) REFERENCES oc__tb_servico(id),
  FOREIGN KEY (id_usuario) REFERENCES oc__tb_usuario(id)
)
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | INT | Identificador único |
| `id_servico` | INT | FK para qual serviço está sendo avaliado |
| `id_usuario` | INT | FK para quem fez a avaliação |
| `nota_preco` | TINYINT | 1-5 estrelas (preço é justo?) |
| `nota_tempo_execucao` | TINYINT | 1-5 estrelas (respeitou prazo?) |
| `nota_higiene` | TINYINT | 1-5 estrelas (limpeza?) |
| `nota_educacao` | TINYINT | 1-5 estrelas (educação?) |
| `comentario` | TEXT | Opinião adicional (opcional) |
| `data_avaliacao` | TIMESTAMP | Quando avaliou |

**Relacionamentos**:
- `id_servico` → `oc__tb_servico(id)` - Serviço sendo avaliado
- `id_usuario` → `oc__tb_usuario(id)` - Quem está avaliando

**Exemplo de dado**:
```json
{
  "id": 1,
  "id_servico": 1,
  "id_usuario": 1,
  "nota_preco": 5,
  "nota_tempo_execucao": 4,
  "nota_higiene": 5,
  "nota_educacao": 5,
  "comentario": "Excelente profissional, muito educado!",
  "data_avaliacao": "2026-01-22 19:08:35"
}
```

---

### 4️⃣ Tabela `oc__tb_categoria`

**Propósito**: Categorias/tipos de serviços disponíveis

```sql
CREATE TABLE oc__tb_categoria (
  id INT PRIMARY KEY AUTO_INCREMENT,
  nome_categoria VARCHAR(50) NOT NULL
)
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | INT | Identificador único |
| `nome_categoria` | VARCHAR(50) | Nome da categoria |

**30+ Categorias pré-cadastradas**:
- Arquiteto(a)
- Armador(a) de Ferragens
- Azulejista / Pisagista
- Bombeiro(a) Hidráulico / Encanador(a)
- Calheiro(a)
- Carpinteiro(a)
- Desentupidor(a)
- Designer de Interiores
- Eletricista
- Engenheiro(a) Civil
- Gesseiro(a)
- Impermeabilizador(a)
- E mais 18 categorias...

---

## 🔗 Diagrama de Relacionamentos

```
┌─────────────────────┐
│  oc__tb_usuario     │
├─────────────────────┤
│ id (PK)            │
│ nome_usuario       │
│ email (UNIQUE)     │
│ senha              │
│ login (UNIQUE)     │
│ telefone           │
│ tipo_usuario       │
│ data_cadastro      │
└────────┬───────────┘
         │
         │ id (prestador)
         │
         ├──────────────────────┐
         │                      │
         ▼                      ▼
    ┌─────────────────┐   ┌──────────────────┐
    │ oc__tb_servico  │   │ oc__tb_avaliacao │
    ├─────────────────┤   ├──────────────────┤
    │ id (PK)        │   │ id (PK)          │
    │ id_usuario (FK)├───┤ id_usuario (FK)  │
    │ titulo         │   │ id_servico (FK)──┤───┐
    │ desc_servico   │   │ nota_preco       │   │
    │ id_categoria   │   │ nota_tempo       │   │
    │ imagem_url     │   │ nota_higiene     │   │
    │ data_cadastro  │   │ nota_educacao    │   │
    └────────┬────────┘   │ comentario       │   │
             │            │ data_avaliacao   │   │
             │            └──────────────────┘   │
             │                                   │
             └───────────────────────────────────┘
                       id (serviço avaliado)

┌──────────────────┐
│ oc__tb_categoria │
├──────────────────┤
│ id (PK)          │
│ nome_categoria   │
└────────┬─────────┘
         │
         │ id_categoria
         │
         ▼
    oc__tb_servico
```

---

## 📈 Consultas Principais

### 1. Listar serviços com nota média

```sql
SELECT 
  s.*,
  u.nome_usuario,
  u.email,
  AVG((a.nota_preco + a.nota_tempo_execucao + a.nota_higiene + a.nota_educacao) / 4) as nota_media,
  COUNT(a.id) as total_avaliacoes
FROM oc__tb_servico s
JOIN oc__tb_usuario u ON s.id_usuario = u.id
LEFT JOIN oc__tb_avaliacao a ON s.id = a.id_servico
GROUP BY s.id
ORDER BY s.data_cadastro DESC;
```

**Por quê JOIN?**
- `INNER JOIN` com usuário: Sempre temos um prestador
- `LEFT JOIN` com avaliação: Serviço pode não ter avaliações ainda
- `GROUP BY` agrupa avaliações do mesmo serviço

### 2. Ver avaliações de um serviço

```sql
SELECT 
  a.*,
  u.nome_usuario
FROM oc__tb_avaliacao a
JOIN oc__tb_usuario u ON a.id_usuario = u.id
WHERE a.id_servico = ?
ORDER BY a.data_avaliacao DESC;
```

### 3. Verificar email/login duplicado

```sql
SELECT * FROM oc__tb_usuario 
WHERE email = ? OR login = ?;
```

### 4. Minhas avaliações

```sql
SELECT 
  a.*,
  s.titulo as nome_servico,
  u.nome_usuario
FROM oc__tb_avaliacao a
JOIN oc__tb_servico s ON a.id_servico = s.id
JOIN oc__tb_usuario u ON s.id_usuario = u.id
WHERE a.id_usuario = ?
ORDER BY a.data_avaliacao DESC;
```

---

## 🔐 Segurança no Banco

### 1. Constraints
- `UNIQUE` em email e login: Previne duplicatas
- `CHECK` em avaliações: Nota sempre entre 1-5
- `NOT NULL`: Campos obrigatórios não vazios
- `FOREIGN KEY`: Integridade referencial

### 2. Tipos de Dados Apropriados
- `VARCHAR(100)` para email, `VARCHAR(50)` para login
- `VARCHAR(255)` para senha (bcryptjs cria hash grande)
- `TINYINT` para notas (ocupa menos espaço que INT)
- `TEXT` para descrição e comentários (ilimitado)

### 3. Timestamps
- `DEFAULT CURRENT_TIMESTAMP`: Registra automaticamente hora
- Não armazenar data em string (ruim para consultas)

---

## 📊 Estatísticas de Dados

### Dado do projeto atual:

| Tabela | Registros | Nota |
|--------|-----------|------|
| oc__tb_usuario | ~5 usuários | Alguns são prestadores |
| oc__tb_servico | ~10 serviços | Alguns com imagens |
| oc__tb_avaliacao | ~3 avaliações | Apenas serviços com clientes |
| oc__tb_categoria | 30 categorias | Pré-cadastradas |

---

## 🔍 Índices Importantes

### Existentes (Chaves Primárias)
```sql
PRIMARY KEY (id)  -- Cada tabela tem autoincrement ID
UNIQUE KEY (email)  -- No oc__tb_usuario
UNIQUE KEY (login)  -- No oc__tb_usuario
```

### Recomendados (para melhor performance)
```sql
-- Esses INDEX ajudam buscas frequentes:
CREATE INDEX idx_usuario_tipo ON oc__tb_usuario(tipo_usuario);
CREATE INDEX idx_servico_usuario ON oc__tb_servico(id_usuario);
CREATE INDEX idx_servico_categoria ON oc__tb_servico(id_categoria);
CREATE INDEX idx_avaliacao_servico ON oc__tb_avaliacao(id_servico);
CREATE INDEX idx_avaliacao_usuario ON oc__tb_avaliacao(id_usuario);
```

---

## 🧪 Fluxo de Dados - Exemplo Completo

### Usuário João se registra e cria um serviço

```sql
-- 1. Inserir usuário
INSERT INTO oc__tb_usuario 
(nome_usuario, email, senha, login, telefone, tipo_usuario, data_cadastro) 
VALUES 
('João Silva', 'joao@email.com', '$2a$10$hash...', 'joao_silva', '11999999999', 'usuario', NOW());
-- Resultado: id = 1

-- 2. João vira prestador (UPDATE)
UPDATE oc__tb_usuario 
SET tipo_usuario = 'prestador' 
WHERE id = 1;

-- 3. João cria serviço
INSERT INTO oc__tb_servico 
(id_usuario, titulo, desc_servico, id_categoria, imagem_url, data_cadastro) 
VALUES 
(1, 'Encanamento', 'Conserto de vazamentos...', 4, 'http://localhost:3001/uploads/imagem.jpg', NOW());
-- Resultado: id = 1

-- 4. Cliente Maria avalia o serviço de João
INSERT INTO oc__tb_avaliacao 
(id_servico, id_usuario, nota_preco, nota_tempo_execucao, nota_higiene, nota_educacao, comentario, data_avaliacao) 
VALUES 
(1, 2, 5, 4, 5, 5, 'Excelente!', NOW());

-- 5. Ver serviço de João com nota média
SELECT 
  s.*,
  u.nome_usuario,
  AVG((a.nota_preco + a.nota_tempo_execucao + a.nota_higiene + a.nota_educacao) / 4) as nota_media
FROM oc__tb_servico s
JOIN oc__tb_usuario u ON s.id_usuario = u.id
LEFT JOIN oc__tb_avaliacao a ON s.id = a.id_servico
WHERE s.id = 1
GROUP BY s.id;
-- Resultado: Serviço com nota 4.75 (média das 4 notas)
```

---

## 🎯 Resumo da Estrutura

| Aspecto | Detalhe |
|---------|---------|
| **Tipo** | Relacional (MySQL) |
| **Tabelas** | 4 principais |
| **Relações** | 1-N (usuário→serviços), 1-N (serviço→avaliações) |
| **Integridade** | FK constraints, UNIQUE, CHECK |
| **Permissões** | Não configuradas (tudo acesso root em dev) |
| **Backup** | SQL dump disponível em `_db/obraconnect_db.sql` |

---

Esta é a base de dados do ObraConnect! Um design simples mas eficiente para um marketplace de serviços.
