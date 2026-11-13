# Capítulo 2 — Criando um menu com NUI

Vamos criar um menu simples que abre com o comando `/menu`, mostra um botão e fecha com o botão ou a tecla ESC.

## Estrutura

~~~
meu-menu/
├── fxmanifest.lua
├── client.lua
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

files {
  'html/index.html',
  'html/style.css',
  'html/script.js',
}

client_script 'client.lua'
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
  <div id="menu" class="oculto">
    <h1>Meu Menu</h1>
    <p id="saudacao"></p>
    <button id="btn-acao">Executar ação</button>
    <button id="btn-fechar">Fechar</button>
  </div>
  <script src="script.js"></script>
</body>
</html>
~~~

## `html/style.css`

~~~css
body {
  margin: 0;
  font-family: Arial, sans-serif;
  background: transparent;      /* a página fica sobre o jogo */
}

.oculto { display: none; }

#menu {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 320px;
  padding: 24px;
  color: #fff;
  text-align: center;
  background: rgba(20, 24, 33, 0.92);
  border-radius: 12px;
}

button {
  margin: 6px;
  padding: 10px 18px;
  border: 0;
  border-radius: 8px;
  cursor: pointer;
  color: #fff;
  background: #2c5364;
}

button:hover { background: #3a7088; }
~~~

## `html/script.js`

~~~javascript
const menu = document.getElementById('menu');

// Função auxiliar para chamar o Lua
function chamarLua(nome, dados = {}) {
  return fetch(${'https://'}${GetParentResourceName()}/${nome}, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json; charset=UTF-8' },
    body: JSON.stringify(dados),
  });
}

// Lua → NUI
window.addEventListener('message', (evento) => {
  const dados = evento.data;

  if (dados.action === 'abrir') {
    document.getElementById('saudacao').textContent = dados.nome;
    menu.classList.remove('oculto');
  }

  if (dados.action === 'fechar') {
    menu.classList.add('oculto');
  }
});

// NUI → Lua
document.getElementById('btn-acao').addEventListener('click', () => {
  chamarLua('acao', { valor: 1 });
});

document.getElementById('btn-fechar').addEventListener('click', () => {
  chamarLua('fechar');
});

// Fechar com ESC
document.addEventListener('keyup', (e) => {
  if (e.key === 'Escape') chamarLua('fechar');
});
~~~

## `client.lua`

~~~lua
local aberto = false

local function alternarMenu(estado)
  aberto = estado
  SetNuiFocus(estado, estado)
  SendNUIMessage({
    action = estado and 'abrir' or 'fechar',
    nome = 'Bem-vindo ao menu!',
  })
end

RegisterCommand('menu', function()
  alternarMenu(not aberto)
end, false)

RegisterNUICallback('fechar', function(data, cb)
  alternarMenu(false)
  cb({ ok = true })
end)

RegisterNUICallback('acao', function(data, cb)
  print('Ação recebida da NUI, valor:', data.valor)
  cb({ ok = true })
end)
~~~

> ⚠️ Todo `RegisterNUICallback` **precisa chamar `cb(...)`**. Se esquecer, a requisição da página fica pendente.

## Como testar

1. Coloque a pasta `meu-menu` em `resources` e adicione `ensure meu-menu` ao `server.cfg`.
2. Entre no jogo e digite `/menu`.
3. Teste o botão "Executar ação" (veja o `print` no F8) e a tecla ESC.

## Problemas comuns

| Sintoma | Causa provável |
| --- | --- |
| Página em branco | Arquivo não listado em `files` ou caminho do `ui_page` errado |
| Mouse não aparece | Faltou `SetNuiFocus(true, true)` |
| Jogador preso na interface | Faltou `SetNuiFocus(false, false)` ao fechar |
| Botão não faz nada | Nome do callback diferente entre JS e Lua |

## Desafios

1. Mude as cores e a fonte do menu.
2. Adicione um campo de texto e envie o valor digitado para o Lua.
3. Faça o Lua responder ao clique com uma mensagem que aparece no menu.

---

⬅️ [Capítulo 1](01-como-a-nui-funciona.md) · ➡️ [Capítulo 3 — MySQL com oxmysql](03-mysql-com-oxmysql.md)
