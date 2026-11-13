# Capítulo 4 — Projeto: caderno de notas

Vamos juntar tudo: uma **interface NUI** onde o jogador vê e adiciona anotações, salvas no **MySQL**.

## Fluxo

~~~
/notas  →  client pede ao server (evento)
        →  server consulta o MySQL
        →  server envia a lista ao client
        →  client abre a NUI com a lista

NUI "Adicionar" → client → server (valida) → MySQL → lista atualizada
~~~

Note que **o server decide tudo**: a NUI só pede e mostra.

## Estrutura

~~~
caderno-de-notas/
├── fxmanifest.lua
├── client.lua
├── server.lua
└── html/
    ├── index.html
    ├── style.css
    └── script.js
~~~

## `fxmanifest.lua`

~~~lua
fx_version 'cerulean'
game 'gta5'
lua54 'yes'

ui_page 'html/index.html'
files { 'html/index.html', 'html/style.css', 'html/script.js' }

client_script 'client.lua'
server_scripts {
  '@oxmysql/lib/MySQL.lua',
  'server.lua',
}
~~~

## `server.lua`

~~~lua
local MAX_NOTAS = 50
local MAX_TEXTO = 200

-- Cria a tabela ao iniciar
MySQL.ready(function()
  MySQL.query.await([[
    CREATE TABLE IF NOT EXISTS notas (
      id INT AUTO_INCREMENT PRIMARY KEY,
      licenca VARCHAR(60) NOT NULL,
      texto VARCHAR(255) NOT NULL,
      criado_em TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      INDEX (licenca)
    )
  ]])
end)

local function obterLicenca(src)
  return GetPlayerIdentifierByType(src, 'license')
end

local function listarNotas(licenca)
  return MySQL.query.await(
    'SELECT id, texto FROM notas WHERE licenca = ? ORDER BY id DESC LIMIT ?',
    { licenca, MAX_NOTAS }
  )
end

-- Cliente pede para abrir o caderno
RegisterNetEvent('notas:abrir', function()
  local src = source
  local licenca = obterLicenca(src)
  if not licenca then return end

  TriggerClientEvent('notas:mostrar', src, listarNotas(licenca))
end)

-- Cliente adiciona uma nota
RegisterNetEvent('notas:adicionar', function(texto)
  local src = source
  local licenca = obterLicenca(src)
  if not licenca then return end

  -- Validação: o client pode enviar qualquer coisa
  if type(texto) ~= 'string' then return end
  texto = texto:gsub('^%s+', ''):gsub('%s+$', '')   -- remove espaços nas pontas
  if #texto == 0 or #texto > MAX_TEXTO then return end

  local total = MySQL.scalar.await(
    'SELECT COUNT(*) FROM notas WHERE licenca = ?', { licenca }
  )
  if total >= MAX_NOTAS then
    TriggerClientEvent('chat:addMessage', src, { args = { 'Notas', 'Limite de notas atingido.' } })
    return
  end

  MySQL.insert.await(
    'INSERT INTO notas (licenca, texto) VALUES (?, ?)',
    { licenca, texto }
  )

  TriggerClientEvent('notas:mostrar', src, listarNotas(licenca))
end)

-- Cliente apaga uma nota (só a própria)
RegisterNetEvent('notas:apagar', function(id)
  local src = source
  local licenca = obterLicenca(src)
  if not licenca or type(id) ~= 'number' then return end

  MySQL.update.await(
    'DELETE FROM notas WHERE id = ? AND licenca = ?',
    { id, licenca }
  )

  TriggerClientEvent('notas:mostrar', src, listarNotas(licenca))
end)
~~~

Pontos de segurança deste código:

- A **licença** vem do server (`GetPlayerIdentifierByType`), nunca do client.
- O texto é **validado** (tipo, tamanho, vazio).
- Há **limite** de notas por jogador.
- O `DELETE` confere o `licenca`: ninguém apaga nota de outra pessoa.
- Todas as consultas usam **parâmetros** (`?`).

## `client.lua`

~~~lua
local aberto = false

local function definirEstado(estado)
  aberto = estado
  SetNuiFocus(estado, estado)
  if not estado then
    SendNUIMessage({ action = 'fechar' })
  end
end

RegisterCommand('notas', function()
  TriggerServerEvent('notas:abrir')
end, false)

-- Server → client: mostra a lista
RegisterNetEvent('notas:mostrar', function(notas)
  if not aberto then definirEstado(true) end
  SendNUIMessage({ action = 'abrir', notas = notas })
end)

-- NUI → client → server
RegisterNUICallback('adicionar', function(data, cb)
  TriggerServerEvent('notas:adicionar', data.texto)
  cb({ ok = true })
end)

RegisterNUICallback('apagar', function(data, cb)
  TriggerServerEvent('notas:apagar', data.id)
  cb({ ok = true })
end)

RegisterNUICallback('fechar', function(_, cb)
  definirEstado(false)
  cb({ ok = true })
end)
~~~

## `html/index.html`

~~~html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div id="caderno" class="oculto">
    <h1>📓 Minhas notas</h1>
    <div class="linha">
      <input id="texto" maxlength="200" placeholder="Escreva uma nota…">
      <button id="btn-add">Adicionar</button>
    </div>
    <ul id="lista"></ul>
    <button id="btn-fechar">Fechar</button>
  </div>
  <script src="script.js"></script>
</body>
</html>
~~~

## `html/script.js`

~~~javascript
const caderno = document.getElementById('caderno');
const lista = document.getElementById('lista');
const campo = document.getElementById('texto');

function chamarLua(nome, dados = {}) {
  return fetch(`https://${GetParentResourceName()}/${nome}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json; charset=UTF-8' },
    body: JSON.stringify(dados),
  });
}

function renderizar(notas) {
  lista.innerHTML = '';
  notas.forEach((nota) => {
    const li = document.createElement('li');
    li.textContent = nota.texto;          // textContent evita injetar HTML

    const btn = document.createElement('button');
    btn.textContent = '🗑️';
    btn.addEventListener('click', () => chamarLua('apagar', { id: nota.id }));

    li.appendChild(btn);
    lista.appendChild(li);
  });
}

window.addEventListener('message', (evento) => {
  const dados = evento.data;
  if (dados.action === 'abrir') {
    renderizar(dados.notas || []);
    caderno.classList.remove('oculto');
  }
  if (dados.action === 'fechar') {
    caderno.classList.add('oculto');
  }
});

document.getElementById('btn-add').addEventListener('click', () => {
  const texto = campo.value.trim();
  if (!texto) return;
  chamarLua('adicionar', { texto });
  campo.value = '';
});

document.getElementById('btn-fechar').addEventListener('click', () => chamarLua('fechar'));
document.addEventListener('keyup', (e) => {
  if (e.key === 'Escape') chamarLua('fechar');
});
~~~

> 💡 O `style.css` pode ser uma adaptação do capítulo 2: fundo escuro, bordas arredondadas e a classe `.oculto { display: none; }`.

> 🔒 Repare em `li.textContent`: nunca use `innerHTML` com texto digitado por jogadores (evita injeção de HTML/JS).

## Como testar

1. Garanta que o `oxmysql` está iniciado e conectado ao banco.
2. `ensure caderno-de-notas`.
3. No jogo, digite `/notas`, adicione, apague e feche com ESC.
4. Reinicie o servidor e abra de novo: as notas continuam lá. ✅

## Desafios

1. Adicione a data da nota (`criado_em`) na interface.
2. Impeça que o jogador envie mais de uma nota por segundo (limite de taxa no server).
3. Permita editar uma nota existente.

## 🎓 Conclusão

Você viu como NUI e MySQL trabalham juntos: a interface mostra, o client repassa, o **server valida e grava**. Com isso você já tem base para criar sistemas como inventários, lojas, registros e painéis.

Veja também a lista de **[recursos úteis](https://github.com/kennedy-hkdv/fivem-recursos-uteis)** para continuar estudando.

---

⬅️ [Capítulo 3](03-mysql-com-oxmysql.md) · 🏠 [Índice](../README.md)
