# 📚 GUIA PASSO A PASSO - 03: CRIAR BANCO DE DADOS

## 🎯 Objetivo

Criar o banco de dados MySQL com todas as tabelas necessárias

---

## 🗄️ Passo 1: Conectar ao MySQL

**Opção 1: Via Terminal**

```powershell
# Abrir terminal (CMD ou PowerShell)
mysql -u root -p

# Pedir senha (deixar em branco se não tem)
# Prompt muda para: mysql>
```

**Opção 2: Via phpMyAdmin (Visual)**

1. Abrir http://localhost/phpmyadmin
2. Fazer login (root, senha em branco se não tem)
3. Abrir aba "SQL" para executar comandos

---

## 🗄️ Passo 2: Criar Banco de Dados

```sql
-- Criar banco
CREATE DATABASE IF NOT EXISTS `obraconnect_db` 
DEFAULT CHARACTER SET utf8mb4 
COLLATE utf8mb4_general_ci;

-- Usar banco
USE `obraconnect_db`;

-- Verificar se criou
SHOW DATABASES;
-- Deve aparecer: obraconnect_db
```

**Por quê UTF8MB4?**

```
UTF8: Padrão texto
- Suporta caracteres normais: abc, 123
- NÃO suporta emojis ou caracteres especiais

UTF8MB4: Estendido
- Suporta caracteres normais: abc, 123
- SUPORTA emojis: 😊, ❤️
- SUPORTA acentos: ñ, ç, ã, é

Escolher UTF8MB4 = Futuro-proof para qualquer idioma
```

---

## 🗄️ Passo 3: Criar Tabela de Usuários

```sql
-- Tabela oc__tb_usuario
CREATE TABLE `oc__tb_usuario` (
  `id` INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  
  `nome_usuario` VARCHAR(100) NOT NULL,
  `email` VARCHAR(100) NOT NULL UNIQUE,
  `senha` VARCHAR(255) NOT NULL,
  `login` VARCHAR(50) NOT NULL UNIQUE,
  `telefone` VARCHAR(20),
  
  `tipo_usuario` ENUM('usuario', 'prestador', 'admin') DEFAULT 'usuario',
  `data_cadastro` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  -- Índices para buscas rápidas
  INDEX idx_email (email),
  INDEX idx_login (login),
  INDEX idx_tipo_usuario (tipo_usuario)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE utf8mb4_general_ci;
```

**Explicação dos campos:**

| Campo | Tipo | Explicação |
|-------|------|-----------|
| `id` | INT AUTO_INCREMENT PRIMARY KEY | ID único, auto incrementa |
| `nome_usuario` | VARCHAR(100) | Nome completo, obrigatório |
| `email` | VARCHAR(100) UNIQUE | Email único para login |
| `senha` | VARCHAR(255) | Hash bcryptjs (precisa espaço) |
| `login` | VARCHAR(50) UNIQUE | Username único |
| `telefone` | VARCHAR(20) | Opcional, contato |
| `tipo_usuario` | ENUM | 'usuario', 'prestador' ou 'admin' |
| `data_cadastro` | TIMESTAMP | Registra data/hora automática |

**Por quê UNIQUE em email e login?**

```sql
-- UNIQUE = Não permite duplicatas
INSERT INTO oc__tb_usuario (email, ...) VALUES ('joao@email.com', ...);
INSERT INTO oc__tb_usuario (email, ...) VALUES ('joao@email.com', ...);
-- Erro: Duplicate entry 'joao@email.com'
-- Bom: Previne contas duplicadas
```

---

## 🗄️ Passo 4: Criar Tabela de Serviços

```sql
-- Tabela oc__tb_servico
CREATE TABLE `oc__tb_servico` (
  `id` INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  
  `id_usuario` INT(11) NOT NULL,
  `titulo` VARCHAR(150) NOT NULL,
  `desc_servico` TEXT,
  `id_categoria` INT(11),
  `imagem_url` VARCHAR(255),
  
  `data_cadastro` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  -- Chaves estrangeiras
  FOREIGN KEY (`id_usuario`) REFERENCES `oc__tb_usuario` (`id`) ON DELETE CASCADE,
  FOREIGN KEY (`id_categoria`) REFERENCES `oc__tb_categoria` (`id`) ON DELETE SET NULL,
  
  -- Índices
  INDEX idx_id_usuario (id_usuario),
  INDEX idx_id_categoria (id_categoria)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE utf8mb4_general_ci;
```

**Explicação de FOREIGN KEY:**

```sql
FOREIGN KEY (id_usuario) REFERENCES oc__tb_usuario (id) ON DELETE CASCADE

Significa:
- id_usuario nesta tabela deve existir em oc__tb_usuario.id
- Se user é deletado, todos seus serviços são deletados
- Garante integridade referencial

Exemplo:
┌────────────────────┐       ┌────────────────────┐
│ oc__tb_servico     │       │ oc__tb_usuario     │
├────────────────────┤       ├────────────────────┤
│ id    │ id_usuario │       │ id                 │
│ 1     │ 1    ─────────────→│ 1 (João)          │
│ 2     │ 1    ─────────────→│                    │
│ 3     │ 2    ────────────┐ │ 2 (Maria)         │
└────────────────────┘    └─────────────────────┘
            ↓
Deletar user 1 (João)
            ↓
serviços 1 e 2 deletam automaticamente
```

---

## 🗄️ Passo 5: Criar Tabela de Avaliações

```sql
-- Tabela oc__tb_avaliacao
CREATE TABLE `oc__tb_avaliacao` (
  `id` INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  
  `id_servico` INT(11) NOT NULL,
  `id_usuario` INT(11) NOT NULL,
  
  `nota_preco` TINYINT(4) NOT NULL CHECK (nota_preco BETWEEN 1 AND 5),
  `nota_tempo_execucao` TINYINT(4) NOT NULL CHECK (nota_tempo_execucao BETWEEN 1 AND 5),
  `nota_higiene` TINYINT(4) NOT NULL CHECK (nota_higiene BETWEEN 1 AND 5),
  `nota_educacao` TINYINT(4) NOT NULL CHECK (nota_educacao BETWEEN 1 AND 5),
  
  `comentario` TEXT,
  `data_avaliacao` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  -- Chaves estrangeiras
  FOREIGN KEY (`id_servico`) REFERENCES `oc__tb_servico` (`id`) ON DELETE CASCADE,
  FOREIGN KEY (`id_usuario`) REFERENCES `oc__tb_usuario` (`id`) ON DELETE CASCADE,
  
  -- Índices
  INDEX idx_id_servico (id_servico),
  INDEX idx_id_usuario (id_usuario)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE utf8mb4_general_ci;
```

**Explicação de CHECK:**

```sql
CHECK (nota_preco BETWEEN 1 AND 5)

Significa:
- nota_preco só pode ser 1, 2, 3, 4 ou 5
- Se tentar inserir 10 ou 0: erro!

INSERT INTO oc__tb_avaliacao (nota_preco, ...) VALUES (5, ...);  -- ✅ OK
INSERT INTO oc__tb_avaliacao (nota_preco, ...) VALUES (10, ...); -- ❌ Erro!
```

---

## 🗄️ Passo 6: Criar Tabela de Categorias

```sql
-- Tabela oc__tb_categoria
CREATE TABLE `oc__tb_categoria` (
  `id` INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `nome_categoria` VARCHAR(50) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE utf8mb4_general_ci;

-- Inserir categorias pré-definidas
INSERT INTO `oc__tb_categoria` (`id`, `nome_categoria`) VALUES
(1, 'Arquiteto(a)'),
(2, 'Armador(a) de Ferragens'),
(3, 'Azulejista / Pisagista'),
(4, 'Bombeiro(a) Hidráulico / Encanador(a)'),
(5, 'Calheiro(a)'),
(6, 'Carpinteiro(a)'),
(7, 'Desentupidor(a)'),
(8, 'Designer de Interiores'),
(9, 'Eletricista'),
(10, 'Engenheiro(a) Civil'),
(11, 'Gesseiro(a)'),
(12, 'Impermeabilizador(a)'),
(13, 'Instalador(a) de Ar Condicionado'),
(14, 'Instalador(a) de Drywall'),
(15, 'Instalador(a) de Gás'),
(16, 'Instalador(a) de Sistemas de Segurança'),
(17, 'Jardineiro(a) / Paisagista'),
(18, 'Limpador(a) Pós-Obra'),
(19, 'Marceneiro(a)'),
(20, 'Marido de Aluguel'),
(21, 'Mestre de Obras'),
(22, 'Montador(a) de Andaimes'),
(23, 'Montador(a) de Móveis'),
(24, 'Terraplanagem'),
(25, 'Pedreiro(a)'),
(26, 'Pintor(a)'),
(27, 'Serralheiro(a)'),
(28, 'Técnico(a) em Edificações'),
(29, 'Topógrafo(a)'),
(30, 'Vidraceiro(a)');
```

---

## 🧪 Passo 7: Verificar Criação

```sql
-- Ver todas as tabelas criadas
SHOW TABLES;
-- Deve mostrar:
-- oc__tb_avaliacao
-- oc__tb_categoria
-- oc__tb_servico
-- oc__tb_usuario

-- Ver estrutura de cada tabela
DESCRIBE oc__tb_usuario;
DESCRIBE oc__tb_servico;
DESCRIBE oc__tb_avaliacao;
DESCRIBE oc__tb_categoria;
```

---

## 📝 Verificar Dados Iniciais

```sql
-- Verificar categorias inseridas
SELECT COUNT(*) FROM oc__tb_categoria;
-- Deve retornar: 30

-- Listar algumas categorias
SELECT * FROM oc__tb_categoria LIMIT 10;
```

---

## 💾 Exportar Banco (Backup)

**Via CLI:**
```powershell
# Exportar todo banco para arquivo SQL
mysqldump -u root -p obraconnect_db > obraconnect_db.sql

# Se não tem senha, usar:
mysqldump -u root obraconnect_db > obraconnect_db.sql
```

**Via phpMyAdmin:**
1. Abrir http://localhost/phpmyadmin
2. Selecionar db `obraconnect_db`
3. Clicar aba "Exportar"
4. Clicar "Enviar" (salva arquivo SQL)

---

## 📋 SQL Completo (Copiar e Colar)

Se preferir, execute tudo de uma vez:

```sql
-- Criar banco
CREATE DATABASE IF NOT EXISTS `obraconnect_db` 
DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;

USE `obraconnect_db`;

-- Tabela usuários
CREATE TABLE `oc__tb_usuario` (
  `id` INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `nome_usuario` VARCHAR(100) NOT NULL,
  `email` VARCHAR(100) NOT NULL UNIQUE,
  `senha` VARCHAR(255) NOT NULL,
  `login` VARCHAR(50) NOT NULL UNIQUE,
  `telefone` VARCHAR(20),
  `tipo_usuario` ENUM('usuario', 'prestador', 'admin') DEFAULT 'usuario',
  `data_cadastro` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_email (email),
  INDEX idx_login (login)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE utf8mb4_general_ci;

-- Tabela categorias
CREATE TABLE `oc__tb_categoria` (
  `id` INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `nome_categoria` VARCHAR(50) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE utf8mb4_general_ci;

-- Inserir categorias
INSERT INTO `oc__tb_categoria` VALUES
(1, 'Arquiteto(a)'),
(2, 'Armador(a) de Ferragens'),
(3, 'Azulejista / Pisagista'),
(4, 'Bombeiro(a) Hidráulico / Encanador(a)'),
(5, 'Calheiro(a)'),
(6, 'Carpinteiro(a)'),
(7, 'Desentupidor(a)'),
(8, 'Designer de Interiores'),
(9, 'Eletricista'),
(10, 'Engenheiro(a) Civil'),
(11, 'Gesseiro(a)'),
(12, 'Impermeabilizador(a)'),
(13, 'Instalador(a) de Ar Condicionado'),
(14, 'Instalador(a) de Drywall'),
(15, 'Instalador(a) de Gás'),
(16, 'Instalador(a) de Sistemas de Segurança'),
(17, 'Jardineiro(a) / Paisagista'),
(18, 'Limpador(a) Pós-Obra'),
(19, 'Marceneiro(a)'),
(20, 'Marido de Aluguel'),
(21, 'Mestre de Obras'),
(22, 'Montador(a) de Andaimes'),
(23, 'Montador(a) de Móveis'),
(24, 'Terraplanagem'),
(25, 'Pedreiro(a)'),
(26, 'Pintor(a)'),
(27, 'Serralheiro(a)'),
(28, 'Técnico(a) em Edificações'),
(29, 'Topógrafo(a)'),
(30, 'Vidraceiro(a)');

-- Tabela serviços
CREATE TABLE `oc__tb_servico` (
  `id` INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `id_usuario` INT(11) NOT NULL,
  `titulo` VARCHAR(150) NOT NULL,
  `desc_servico` TEXT,
  `id_categoria` INT(11),
  `imagem_url` VARCHAR(255),
  `data_cadastro` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (`id_usuario`) REFERENCES `oc__tb_usuario` (`id`) ON DELETE CASCADE,
  FOREIGN KEY (`id_categoria`) REFERENCES `oc__tb_categoria` (`id`) ON DELETE SET NULL,
  INDEX idx_id_usuario (id_usuario),
  INDEX idx_id_categoria (id_categoria)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE utf8mb4_general_ci;

-- Tabela avaliações
CREATE TABLE `oc__tb_avaliacao` (
  `id` INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `id_servico` INT(11) NOT NULL,
  `id_usuario` INT(11) NOT NULL,
  `nota_preco` TINYINT(4) NOT NULL CHECK (nota_preco BETWEEN 1 AND 5),
  `nota_tempo_execucao` TINYINT(4) NOT NULL CHECK (nota_tempo_execucao BETWEEN 1 AND 5),
  `nota_higiene` TINYINT(4) NOT NULL CHECK (nota_higiene BETWEEN 1 AND 5),
  `nota_educacao` TINYINT(4) NOT NULL CHECK (nota_educacao BETWEEN 1 AND 5),
  `comentario` TEXT,
  `data_avaliacao` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (`id_servico`) REFERENCES `oc__tb_servico` (`id`) ON DELETE CASCADE,
  FOREIGN KEY (`id_usuario`) REFERENCES `oc__tb_usuario` (`id`) ON DELETE CASCADE,
  INDEX idx_id_servico (id_servico),
  INDEX idx_id_usuario (id_usuario)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE utf8mb4_general_ci;
```

---

## ✅ Checklist

- [ ] Banco `obraconnect_db` criado
- [ ] Tabela `oc__tb_usuario` criada
- [ ] Tabela `oc__tb_categoria` criada e populada com 30 categorias
- [ ] Tabela `oc__tb_servico` criada
- [ ] Tabela `oc__tb_avaliacao` criada
- [ ] Todas as foreign keys configuradas
- [ ] Banco testado em `localhost:3001/teste-banco`

---

✅ **Banco de dados pronto!** Próximo passo: Implementar autenticação com JWT e bcryptjs.
