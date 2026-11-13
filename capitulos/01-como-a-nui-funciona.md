# Capítulo 1 — Como a NUI funciona

**NUI** (*Native User Interface*) é o sistema do FiveM que permite exibir **páginas web** (HTML, CSS e JavaScript) por cima do jogo. É assim que são feitos menus, inventários, HUDs, telas de login e quase toda interface moderna.

## A ideia

~~~
┌──────────────┐   SendNUIMessage    ┌──────────────┐
│   Lua        │ ──────────────────► │  HTML/JS     │
│  (client)    │                     │  (NUI)       │
│              │ ◄────────────────── │              │
└──────────────┘   fetch (callback)  └──────────────┘
~~~

- **Lua → NUI:** `SendNUIMessage` envia dados para a página.
- **NUI → Lua:** a página faz uma requisição (`fetch`) que é recebida por `RegisterNUICallback`.

## Peças principais

| Peça | Onde | Função |
| --- | --- | --- |
| `ui_page` | `fxmanifest.lua` | Define a página HTML principal |
| `files` | `fxmanifest.lua` | Lista os arquivos (HTML, CSS, JS, imagens) que o client deve baixar |
| `SendNUIMessage` | Lua (client) | Envia uma tabela (vira JSON) para a página |
| `SetNuiFocus` | Lua (client) | Liga/desliga mouse e teclado para a interface |
| `RegisterNUICallback` | Lua (client) | Recebe chamadas vindas da página |
| `GetParentResourceName()` | JavaScript | Nome do resource, usado para montar a URL do callback |

## O foco (`SetNuiFocus`)

~~~lua
SetNuiFocus(true, true)    -- (teclado, mouse): o jogador interage com a interface
SetNuiFocus(false, false)  -- devolve o controle ao jogo
~~~

Se você esquecer de devolver o foco, o jogador fica **preso** na interface. Sempre prepare uma forma de fechar (botão e tecla ESC).

## Regras importantes

1. A interface roda no **client**. Nunca coloque regra de negócio só no JavaScript — ele pode ser alterado pelo jogador.
2. Valide tudo no **server** (capítulo 4 do ebook anterior).
3. Mantenha a página **leve**: imagens otimizadas, sem bibliotecas desnecessárias.
4. Esconda a interface por padrão e mostre só quando o Lua mandar.

## Checklist do capítulo

- [ ] Sei o que é NUI e para que serve
- [ ] Entendo as duas direções de comunicação (Lua → NUI e NUI → Lua)
- [ ] Sei o que `SetNuiFocus` faz

---

🏠 [Índice](../README.md) · ➡️ [Capítulo 2 — Criando um menu com NUI](02-criando-um-menu-nui.md)
