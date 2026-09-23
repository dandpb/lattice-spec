# Lattice — especificação para reproduzir

Construa **Lattice**, um Game of Life de Conway no navegador, dark, com grid pintável, paleta por idade e padrões clássicos.

O universo é um torus de **240×140**. Regras **B3/S23**: célula viva com 2 ou 3 vizinhos sobrevive; morta com exatamente 3 nasce; o resto morre. Vizinhos fora do mundo só existem se Wrap estiver ligado (borda = toro). Sem login, sem banco. Estado só em memória + `localStorage`.

Não simplifique para “quadrados verdes”. A cor por idade, o rastro e o tile arredondado são o produto.

## Visual

Chrome escuro, sem roxo de marca. Fontes IBM Plex Sans e Mono. Alvos de toque ≥ 44px, sem overflow horizontal em 390px. Título da página: **Lattice**.

| Token | Valor |
|---|---|
| Fundo | `#0a0b0c` |
| Surface | `#141618` |
| Surface 2 | `#1c1f22` |
| Texto | `#e8ece9` |
| Muted | `#8a9290` |
| Borda | `#2a2e2c` |

O mundo é um `<canvas>` 2D (não CSS grid). O buffer lógico fica fora do React; o React só desenha o chrome. Loop em `requestAnimationFrame`, timestep fixo: acumula `dt` (cap 0.1 s) e dá no máximo 12 gerações por frame. Velocidade = gerações por segundo (slider **1–60**, default **10**). Enquanto o guia estiver aberto, a simulação pausa.

### Células vivas — cor por idade

Não use um verde só.

- `t = 0` se idade ≤ 1; senão `min(0.88, log2(idade) / 6)`
- 8 faixas de idade: 1, 3, 8, 16, 32, 64, 128, 255
- Paletas interpolam 5 stops nesse `t`
- **Prism** (default) não usa os stops no mundo: HSL `hue 48 + t*280`, saturação `78 − t*18`, luminosidade `72 − t*26`
- Ao morrer, `heat = 140` e decai 28 por geração. Rastro dim, mesma família da paleta, alpha no máximo ~0.34, só se heat ≥ 28
- Células antigas **não** podem escurecer até sumir no fundo morto

### Paletas

Stops são nascimento → velho. O swatch da UI usa os stops; Prism no canvas usa HSL.

| Id | Nome | Hint | Morto RGB | Heat RGB | Stops RGB |
|---|---|---|---|---|---|
| `celadon` | Celadon | Mint births, teal-blue still-lifes | 16, 18, 19 | 255, 210, 140 | 248,255,252 · 140,230,186 · 64,196,168 · 48,158,176 · 72,128,196 |
| `ember` | Ember | Hot sparks that cool to coal | 22, 14, 12 | 255, 196, 90 | 255,247,214 · 255,179,71 · 232,93,4 · 200,70,40 · 168,52,48 |
| `tide` | Tide | Ice births, cobalt still-lifes | 12, 16, 22 | 180, 230, 255 | 236,252,255 · 125,211,252 · 56,189,248 · 37,99,235 · 67,56,202 |
| `aurora` | Aurora | Teal to violet by age | 14, 14, 22 | 244, 114, 182 | 94,234,212 · 56,189,248 · 167,139,250 · 232,121,249 · 244,114,182 |
| `prism` | Prism | Full spectrum by longevity | 14, 14, 16 | 251, 191, 36 | 254,240,138 · 74,222,128 · 34,211,238 · 167,139,250 · 244,114,182 |

Fundo fora do mundo:

| Paleta | Outside |
|---|---|
| Celadon | `#070808` |
| Ember | `#0a0706` |
| Tide | `#05080c` |
| Aurora | `#07080c` |
| Prism | `#08080a` |

### Render

- Zoom `< 4px`: `ImageData` nearest-neighbor (LUT de idade + heat), sem grid
- `≥ 4px`: tiles arredondados, raio `max(3, round(cellSize * 0.4))`; highlight branco ~42% no topo a partir de 6px
- Grid de 1px só a partir de **8px**
- Gap de 1px entre células se a face ≥ 10px
- Retângulos com `Math.round` para não borrar
- Zoom de 1 a 40px; a partir de 4px, arredonda para inteiro
- Zoom vai em direção ao cursor
- **Fit** encaixa o mundo com 16px de margem. No primeiro encaixe, se a célula ficar `< 12px`, força **12px** e mostra o canto do canhão (`panX 0`, `panY 8`)
- Legenda **New → Old** no canto inferior esquerdo, também no telefone

## Motor

Três `Uint8Array` do tamanho do mundo:

- `cells` — 0/1. A soma de vizinhos **só** usa isso
- `age` — 0–255
- `heat` — rastro de morte

Double buffer em `step`.

- Pintar uma célula morta: `age = 1`, heat 0
- Apagar: age 0, heat 140
- **Randomize**: densidade 0.22, zera geração, heat 0, idade aleatória `1 + random * 80` nas vivas (a sopa já nasce colorida)
- Clear zera células, idade, heat, população e geração

## Pintura

Pointer events, não mouse puro.

- O primeiro clique decide: se a célula está viva, o traço apaga; se morta, pinta. Shift ou botão direito força apagar
- O traço interpola com **Bresenham** + `getCoalescedEvents`, senão o arrasto fura
- Pincel 1, 2 ou 3: quadrado de Chebyshev, raio `brush - 1`
- Pan: Alt, botão do meio, tecla Hand (`H`), ou dois dedos
- Pinch: dois dedos, zoom pela distância, deadzone ~3.5%
- Roda do mouse faz zoom
- Carimbo: o padrão segue o cursor (ghost semitransparente, também arredondado). Clique ou Enter solta centrado. Escape cancela. Carimbo ganha do modo pan

## Demo inicial

Carimbar com origem no canto superior esquerdo do padrão. Começa **rodando**.

| Padrão | Origem (x, y) |
|---|---|
| Gosper glider gun | 12, 22 |
| Pulsar | 168, 18 |
| Glider | 108, 88 |
| Lightweight spaceship | 42, 108 |

## Padrões

Plaintext: `O` = viva, `.` = morta.

Grupos e dicas do menu:

| Grupo | Dica |
|---|---|
| Still life | Never change |
| Oscillator | Cycle in place |
| Spaceship | Move each generation |
| Gun | Emit spaceships forever |
| Methuselah | Grow for a long time |

### Still life

**Block**

```
OO
OO
```

**Beehive**

```
.OO
O..O
.OO
```

**Loaf**

```
.OO.
O..O
.O.O
..O.
```

**Boat**

```
OO.
O.O
.O.
```

**Ship**

```
OO.
O.O
.OO
```

### Oscillator

**Blinker**

```
OOO
```

**Toad**

```
.OOO
OOO.
```

**Beacon**

```
OO..
OO..
..OO
..OO
```

**Clock** — oscilador período 2, não still life

```
..O.
O.O.
.O.O
.O..
```

**Pentadecathlon**

```
..O....O..
OO.OOOO.OO
..O....O..
```

**Pulsar** — 13×13

```
..OOO...OOO..
.............
O....O.O....O
O....O.O....O
O....O.O....O
..OOO...OOO..
.............
..OOO...OOO..
O....O.O....O
O....O.O....O
O....O.O....O
.............
..OOO...OOO..
```

### Spaceship

**Glider**

```
.O.
..O
OOO
```

**Lightweight spaceship**

```
.O..O
O....
O...O
OOOO.
```

**Middleweight spaceship**

```
...O.
.O...O
O.....
O....O
OOOOO.
```

**Heavyweight spaceship**

```
...OO.
.O....O
O......
O.....O
OOOOOO.
```

### Gun

**Gosper glider gun** — 36×9 canônico

```
........................O...........
......................O.O...........
............OO......OO............OO
...........O...O....OO............OO
OO........O.....O...OO..............
OO........O...O.OO....O.O...........
..........O.....O.......O...........
...........O...O....................
............OO......................
```

**Simkin glider gun** — exatamente estas 15 linhas de 33 colunas (29 células, dois barris; tem que disparar)

```
OO.....OO........................
OO.....OO........................
.................................
....OO...........................
....OO...........................
.................................
.................................
.................................
.................................
......................OO.OO......
.....................O.....O.....
.....................O......O..OO
.....................OOO...O...OO
..........................O......
```

Depois de ~30 gerações o Simkin deve crescer (população sobe e gliders saem). Clock deve oscilar período 2, não ficar parado.

### Methuselah

**R-pentomino**

```
.OO
OO.
.O.
```

**Acorn**

```
.O.....
...O...
OO..OOO
```

**Diehard**

```
......O.
OO......
.O...OOO
```

**Thunderbird**

```
OOO
...
.O.
.O.
.O.
```

## Chrome

Barra fixa no topo, fundo quase opaco.

**Linha 1:** marca (glider de 5 quadrados em SVG) + **LATTICE** + subtítulo “Conway's Game of Life” (some no telefone) + **GEN** e **ALIVE** em mono + swatch do look (sempre visível, inclusive no telefone) + botão de guia.

**Linha 2** (ou à direita em desktop largo, sem quebrar): Play/Pause, Step, slider `N/s`, Patterns, e no desktop Pan, Clear, Randomize, pincéis 1 2 3, zoom −/+, Fit, Wrap on/off. No telefone isso vai num menu **More**, com Look no topo do menu também.

Botões 44px, `shrink-0`. Slider com thumb grande e `aria-label`.

Persistência:

| Chave | Valor |
|---|---|
| `lattice-look` | id da paleta (`prism` default) |
| `lattice-tour` | `"1"` depois de Skip ou fechar o guia |

O guia abre na primeira visita.

## Atalhos

| Tecla | Ação |
|---|---|
| Space | Play / pause |
| `N` ou `.` | Um passo e pausa |
| `C` | Limpa |
| `R` | Randomiza |
| `F` | Fit |
| `H` | Alterna pan / pintar |
| `1` `2` `3` | Pincel |
| `P` | Cicla paleta |
| `+` / `−` | Zoom |
| WASD e setas | Pan ~420 px/s |
| Enter | Solta o carimbo no centro da vista |
| Esc | Cancela o carimbo |
| `?` | Abre o guia |

Atalhos não disparam dentro de input. Com o guia aberto, só as teclas do guia funcionam.

## Guia

6 passos, dialog no centro, fundo escurecido. Skip no passo 1; Back depois; Next até fechar. Esc fecha. Setas trocam passo. Enter avança.

1. **The rules** — texto B3/S23 + mini diagrama 3×3 do glider
2. **Paint the grid** — arrastar desenha; o primeiro clique escolhe pintar ou apagar; Shift ou botão direito apaga
3. **Run time** — Space corre; N dá um passo e pausa; slider é gerações por segundo
4. **Stamp a pattern** — Patterns, depois toque na grade; Esc cancela
5. **Move the view** — scroll ou pinch zoom; Alt-drag, dois dedos ou Hand pan; Fit mostra o mundo; Wrap = toro
6. **You're set** — Gen é o tick; Alive é a população; o `?` reabre o guia

## Dicas de rodapé

- Telefone: `Drag to paint · two fingers pan · pinch zoom`
- Desktop: `Paint to draw · right-click or Shift erases · scroll zooms · Alt-drag or Hand pans · Space plays`

## Critério de pronto

- Canhão de Gosper dispara
- Simkin dispara (população sobe depois de ~30 gerações)
- Clock é período 2, não still life
- Pintar com o mouse não fura células
- A 12px as células são pastilhas com canto arredondado (não quadrado 90°) e reflexo no topo
- Still life escurece mas continua visível; glider fica claro; morte deixa rastro curto, sem virar tapete
- Prism, Ember, Tide e Aurora são distinguíveis numa sopa aleatória (Tide azul, Aurora com magenta, Ember quente)
- Telefone 390px: sem scroll horizontal, Look e legenda New→Old visíveis, alvos ≥ 44px
- Sem erros de console
