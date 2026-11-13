# Capítulo 3 — MySQL com oxmysql

Variáveis na memória somem quando o servidor reinicia. Para **guardar dados de forma permanente** (notas, dinheiro, itens, registros), usamos um banco de dados. O mais comum no FiveM é o **MySQL / MariaDB**, acessado pelo resource **oxmysql**.

## O que você precisa

1. Um servidor MySQL ou MariaDB rodando (local ou remoto).
2. Um banco de dados criado, com usuário e senha.
3. O resource **oxmysql** instalado na pasta `resources`.

## Configuração

No `server.cfg`, **antes** dos resources que usam o banco:

~~~cfg
set mysql_connection_string "mysql://usuario:senha@localhost/nome_do_banco?charset=utf8mb4"
ensure oxmysql
ensure meu-script
~~~

> 🔒 Nunca publique o `server.cfg` com a senha do banco no GitHub.

No `fxmanifest.lua` do seu resource, importe a biblioteca **antes** dos seus scripts de servidor:

~~~lua
server_scripts {
  '@oxmysql/lib/MySQL.lua',
  'server.lua',
}
~~~

## Noções rápidas de SQL

~~~sql
-- Criar uma tabela
CREATE TABLE IF NOT EXISTS jogadores (
  id INT AUTO_INCREMENT PRIMARY KEY,
  licenca VARCHAR(60) NOT NULL UNIQUE,
  nome VARCHAR(60) NOT NULL,
  dinheiro INT NOT NULL DEFAULT 0
);

-- Inserir
INSERT INTO jogadores (licenca, nome) VALUES ('license:abc', 'Ana');

-- Consultar
SELECT * FROM jogadores WHERE licenca = 'license:abc';

-- Atualizar
UPDATE jogadores SET dinheiro = 500 WHERE id = 1;

-- Apagar
DELETE FROM jogadores WHERE id = 1;
~~~

## Usando o oxmysql no Lua

Todas as funções aceitam **parâmetros com `?`** (consultas preparadas). A versão `.await` espera o resultado e deve ser usada dentro de uma thread ou de um evento.

| Função | Retorna | Use para |
| --- | --- | --- |
| `MySQL.query.await` | lista de linhas | `SELECT` de várias linhas |
| `MySQL.single.await` | uma linha (ou `nil`) | buscar um registro |
| `MySQL.scalar.await` | um valor | `COUNT`, um único campo |
| `MySQL.insert.await` | ID inserido | `INSERT` |
| `MySQL.update.await` | linhas afetadas | `UPDATE` e `DELETE` |

### Exemplos

~~~lua
-- Buscar um jogador
local jogador = MySQL.single.await(
  'SELECT * FROM jogadores WHERE licenca = ?',
  { licenca }
)

-- Listar todos
local todos = MySQL.query.await('SELECT id, nome FROM jogadores')
for _, j in ipairs(todos) do
  print(j.id, j.nome)
end

-- Inserir
local novoId = MySQL.insert.await(
  'INSERT INTO jogadores (licenca, nome) VALUES (?, ?)',
  { licenca, nome }
)

-- Atualizar
MySQL.update.await(
  'UPDATE jogadores SET dinheiro = dinheiro + ? WHERE id = ?',
  { 100, novoId }
)
~~~

Criar a tabela ao iniciar o resource:

~~~lua
MySQL.ready(function()
  MySQL.query.await([[
    CREATE TABLE IF NOT EXISTS jogadores (
      id INT AUTO_INCREMENT PRIMARY KEY,
      licenca VARCHAR(60) NOT NULL UNIQUE,
      nome VARCHAR(60) NOT NULL,
      dinheiro INT NOT NULL DEFAULT 0
    )
  ]])
end)
~~~

## 🔒 Segurança: nunca monte SQL com concatenação

❌ **Errado** (vulnerável a *SQL injection*):

~~~lua
MySQL.query.await("SELECT * FROM jogadores WHERE nome = '" .. nome .. "'")
~~~

✅ **Certo** (parâmetro com `?`):

~~~lua
MySQL.query.await('SELECT * FROM jogadores WHERE nome = ?', { nome })
~~~

## Boas práticas

1. **Consulte só o que precisa** (evite `SELECT *` em tabelas grandes).
2. **Crie índices** em colunas muito pesquisadas (por exemplo, `licenca`).
3. **Não consulte o banco a cada frame.** Faça consultas em eventos e guarde em memória o que for acessado com frequência.
4. **Valide os dados** recebidos do client antes de gravar.
5. Faça **backup** do banco regularmente.

## Desafios

1. Crie a tabela `jogadores` e um comando de server que cadastra quem entra no servidor (use a licença como identificador).
2. Crie uma função que soma dinheiro a um jogador com `UPDATE`.
3. Conte quantos jogadores já se cadastraram usando `MySQL.scalar.await`.

---

⬅️ [Capítulo 2](02-criando-um-menu-nui.md) · ➡️ [Capítulo 4 — Projeto: caderno de notas](04-projeto-caderno-de-notas.md)
