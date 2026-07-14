# qbx_properties — Manual

Tela de escolha do apartamento inicial: um quadro de seleção em scaleform que o novo personagem usa para escolher onde vai morar; a criação da propriedade é delegada ao `ps-housing`.

---

## Sumário

1. [Estado do recurso](#estado-do-recurso)
2. [Dependências](#dependências)
3. [Instalação](#instalação)
4. [Configuração](#configuração)
5. [Fluxo de uso](#fluxo-de-uso)
6. [Controles](#controles)
7. [Banco de dados](#banco-de-dados)
8. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
9. [Localização](#localização)
10. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Estado do recurso

Este é um fork em que o `fxmanifest.lua` carrega **apenas** `client/apartmentselect.lua`. Todos os demais scripts (o restante do `client/` e o `server/` inteiro) estão comentados no manifesto e **não rodam**:

| Arquivo | Situação |
|---|---|
| `client/apartmentselect.lua` | Ativo — é o único script carregado |
| `client/property.lua` | Comentado no `fxmanifest.lua` |
| `client/realtor.lua` | Comentado no `fxmanifest.lua` |
| `client/dataview.lua` | Comentado no `fxmanifest.lua` |
| `client/decorating.lua` | Comentado no `fxmanifest.lua` |
| `server/*` (bloco inteiro) | Comentado no `fxmanifest.lua` |

Na prática, o recurso hoje entrega só a tela de seleção de apartamento e repassa a criação para o `ps-housing`. Compra, aluguel, corretor, decoração e stashes ficam por conta do `ps-housing` — as tabelas e o config correspondentes seguem no repositório para uso futuro.

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `ox_lib` | Sim | Locale, `lib.requestModel`, `lib.requestScaleformMovie`, `lib.alertDialog`, `cache` |
| `qbx_core` | Sim | Módulo `lib` e os eventos `QBCore:Server:OnPlayerLoaded` / `QBCore:Client:OnPlayerLoaded` disparados após a escolha |
| `ps-housing` | Sim | Fornece a lista de apartamentos (`ps-housing:setApartments`) e cria a propriedade (`ps-housing:server:createNewApartment`) |
| `oxmysql` | Não | Só volta a ser necessário se os scripts de servidor forem reativados |

---

## Instalação

1. Copie a pasta `qbx_properties` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure qbx_properties
   ```
3. Alguém precisa disparar `apartments:client:setupSpawnUI` no cliente do jogador novo (no fluxo atual, o `ps-housing`). O script de servidor que fazia isso está desativado.
4. Os SQLs (`property.sql`, `decorations.sql`) **não** são necessários com o manifesto atual — só importe se reativar os scripts de servidor.
5. **Conflitos** — o README do upstream orienta remover `qbx_apartments` e `qbx_houses` de uma build limpa do QBox.

---

## Configuração

### `config/shared.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `shellUndergroundOffset` | number | Sim | Deslocamento em Z aplicado por `CalculateOffsetCoords` para shells enterrados. Padrão: `50.0` |
| `apartmentOptions` | array | Sim | Opções mostradas no quadro de seleção. Cada entrada tem `interior` (chave em `interiors`), `label`, `description` e `enter` (vec3 do ponto de entrada) |
| `interiors` | tabela | Sim | Por interior: `exit`, `clothing`, `stash` e `logout` (coordenadas dos pontos de interação). Só é lido pelos scripts de servidor, hoje desativados |

> O quadro monta a tela em blocos de 3 opções e as ramificações do código cobrem no máximo 6 entradas em `apartmentOptions`. Passar disso quebra a navegação.

### `config/client.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `furniture` | tabela | Sim | Catálogo de móveis por categoria (`lighting`, `couches`, `tables`, `beds`), cada item com `object` (modelo) e `label`. Consumido apenas por `client/decorating.lua`, hoje desativado |

### `config/server.lua`

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `apartmentStash.slots` | number | Sim | Slots do baú do apartamento. Padrão: `50` |
| `apartmentStash.maxWeight` | number | Sim | Peso máximo do baú, em gramas. Padrão: `150000` |

Este arquivo só é lido pelos scripts de servidor, hoje desativados.

---

## Fluxo de uso

1. Ao receber `apartments:client:setupSpawnUI`, o cliente teleporta e congela o personagem em frente ao quadro, monta a câmera e carrega o scaleform `AUTO_SHOP_BOARD` (streamado por este recurso).
2. Se `apartmentOptions` tiver apenas **uma** opção, a seleção é pulada e o apartamento é confirmado direto.
3. O jogador navega pelas opções e confirma; um `lib.alertDialog` pede a confirmação final.
4. Confirmado: a tela escurece, o personagem é movido para o ponto de entrada do apartamento e o cliente dispara `ps-housing:server:createNewApartment` com o `label` do apartamento escolhido.
5. Em seguida, `QBCore:Server:OnPlayerLoaded` e `QBCore:Client:OnPlayerLoaded` são disparados para o jogador terminar de carregar (roupas, etc.).

A lista efetivamente usada na navegação e na confirmação vem do evento `ps-housing:setApartments`; os textos exibidos no quadro (label e descrição) vêm de `sharedConfig.apartmentOptions`. Mantenha as duas listas na mesma ordem.

---

## Controles

Válidos apenas enquanto o quadro de seleção está aberto.

| Ação | Controle | Tecla padrão |
|---|---|---|
| Opção anterior | 188 | Seta para cima |
| Próxima opção | 187 | Seta para baixo |
| Confirmar | 191 | `Enter` |

---

## Banco de dados

Os SQLs abaixo pertencem aos scripts de servidor desativados. Só importe se for reativá-los.

| Arquivo | Tabela | Conteúdo |
|---|---|---|
| `property.sql` | `properties` | Nome, coords, preço, dono (`citizenid`), interior, keyholders, intervalo de aluguel, pontos de interação e stashes |
| `decorations.sql` | `properties_decorations` | Móveis colocados numa propriedade (modelo, coords, rotação), com `ON DELETE CASCADE` a partir de `properties` |

---

## Entrypoints para outros recursos

### Evento `apartments:client:setupSpawnUI`

Abre o quadro de seleção de apartamento no cliente do jogador. É o ponto de entrada do recurso.

```lua
-- servidor
TriggerClientEvent('apartments:client:setupSpawnUI', playerSource)
```

### Evento `ps-housing:setApartments`

O recurso escuta este evento para receber a lista de apartamentos usada na navegação e na confirmação.

```lua
-- cliente
TriggerEvent('ps-housing:setApartments', apartments)
```

### Função global `CalculateOffsetCoords` (shared)

Aplica um offset às coordenadas de uma propriedade, descontando `shellUndergroundOffset` do Z.

```lua
local coords = CalculateOffsetCoords(propertyCoords, offset) -- retorna vec4
```

---

## Localização

Strings via `ox_lib` locale, em `locales/`:

`en`, `pt-br`, `pt`, `es`, `fr`, `de`, `nl`, `da`, `ro`

Idioma ativo pela convar:

```
setr ox:locale "pt-br"
```

Com o manifesto atual, apenas as chaves `instructButtons.*` e `alert.apartment_selection` / `alert.are_you_sure` são usadas; as demais pertencem aos scripts desativados.

---

## Estrutura de arquivos

```
qbx_properties/
├── client/
│   ├── apartmentselect.lua   — ATIVO: quadro de seleção, câmera, scaleform e confirmação
│   ├── property.lua          — desativado no fxmanifest: entrar/sair, chaves, campainha, menu de gestão
│   ├── realtor.lua           — desativado no fxmanifest: criação de propriedades pelo corretor
│   ├── decorating.lua        — desativado no fxmanifest: colocação de móveis com gizmo
│   └── dataview.lua          — desativado no fxmanifest: helper de leitura binária
├── server/
│   ├── apartmentselect.lua   — desativado: inseria a propriedade em `properties` e criava o stash
│   ├── property.lua          — desativado: entrada/saída, keyholders, stashes
│   ├── realtor.lua           — desativado: criação/venda de propriedades
│   └── decorating.js         — desativado: usado só para gerar os screenshots dos móveis
├── config/
│   ├── shared.lua            — apartmentOptions, interiors, shellUndergroundOffset
│   ├── client.lua            — catálogo de móveis
│   └── server.lua            — slots e peso do baú do apartamento
├── shared/
│   └── main.lua              — CalculateOffsetCoords
├── stream/
│   ├── auto_shop_board.gfx   — scaleform AUTO_SHOP_BOARD do quadro de seleção
│   ├── auto_shop_board.ytd
│   └── auto_shop_board_img.ytd
├── screenshots/              — imagens dos móveis (usadas pela UI de decoração desativada)
├── locales/
│   ├── en.json
│   └── da / de / es / fr / nl / pt / pt-br / ro .json
├── property.sql              — tabela `properties`
├── decorations.sql           — tabela `properties_decorations`
└── fxmanifest.lua
```
