# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é este projeto

Painel de análise de consumo de bebidas alcoólicas para a **Deixa Comigo Bebidas**, divisão Nitro.
Entrega única: `Dashboards/index.html` — um arquivo HTML autocontido (~200 KB) com CSS e JS inline.
Não é um repositório git, não há `package.json`, build, linter ou suíte de testes. O idioma do
código, dos comentários e da interface é **português do Brasil**.

```
Dados/drinks.csv                          amostra de entrada (country, beer/spirit/wine/total)
Referencias/nitro_brand_book_by_pomelli.pdf  brandbook (vetorial, sem texto extraível)
Dashboards/index.html                     o produto
```

## Como rodar / testar

Não há comando de build. Abra o arquivo direto no navegador:

```bash
start "" "Dashboards/index.html"
```

O painel abre vazio de propósito. Para exercitá-lo, arraste `Dados/drinks.csv` para a janela
ou use o botão **Importar planilha**. Teste sempre nos **dois temas** (botão no cabeçalho) —
regressões de contraste no tema claro já aconteceram mais de uma vez.

Sob `file://` o `fetch()` é bloqueado, então o auto-load de `../Dados/drinks.csv` falha em
silêncio (por design). Para exercitar esse caminho, sirva por HTTP a partir da raiz do projeto:

```bash
python -m http.server 8080
```

Nem Python nem Node estão garantidos nesta máquina — Windows com PowerShell 5.1 é o único
ambiente certo. Scripts auxiliares foram escritos em PowerShell por esse motivo.

## Invariantes (não quebre)

1. **Nenhum dado de consumo pode ser hard-coded.** Todo número vem do CSV importado em tempo de
   execução. A única exceção aceita é metadado *geográfico*: listas de continentes, aliases de
   nomes de país e a malha de fronteiras. Se você embutir um valor de consumo para "testar",
   remova antes de salvar.
2. **Arquivo único, sem dependência de CDN para funcionar.** Nada de d3, nada de bundler, nada
   de `fetch()` em runtime. A fonte do Google é o único recurso externo e tem fallback de stack.
3. **Nenhuma chave de API no código.** Elas vivem no `.env` da raiz (git-ignorado), lido em
   runtime por `ENV.load()`. Não crie `config.js`. Sob `file://` o `.env` não é legível — o
   chat pede a chave uma vez e guarda em `localStorage`; a previsão do tempo só não aparece.
4. **Mínimo de cliques até o dado aparecer.** Importou → tudo renderiza. Não acrescente etapas
   de configuração entre o upload e o resultado.

## Arquitetura de `Dashboards/index.html`

O arquivo é montado em quatro blocos, na ordem, cada um com um banner de comentário:

| Bloco | Conteúdo |
|---|---|
| `<style>` | tokens de marca em custom properties + todo o layout |
| markup | cabeçalho, empty state/dropzone, barra de filtros, grade de KPIs, 7 cards, tooltip, toast |
| `<script id="world-geometry">` | `WORLD_TOPO` — TopoJSON Natural Earth 110m, **só geometria** |
| `<script>` | núcleo (parsing/geo/projeção/estatística) seguido da camada de estado/render |

Os banners `/* ==== DADOS ==== */`, `CONTROLES`, `RENDER`, `CHAVES (.env)`, `PREVISÃO DO TEMPO`,
`CHAT COM OS DADOS (Gemini)` e `BOOT` delimitam a segunda metade do JS.

### Integrações externas (chat e clima)

- **`ENV`** — parser de `.env` (`KEY=valor`, `#` comenta, aspas removidas). `ENV.load()` tenta
  `../.env` e depois `.env`; `ENV.get(k)` cai para `localStorage` (`nitro-env-<KEY>`) quando o
  fetch falha. Placeholders `cole-sua-chave…` são ignorados de propósito.
- **`chatContext()`** — monta o prompt a partir de `filtered()`, nunca de `APP.ds.rows`. Envia
  filtros ativos, `describe()` por métrica, médias por continente, correlações e a tabela da
  seleção (acima de 180 países, só os 90 maiores + 90 menores; as estatísticas cobrem todos).
  Toda pergunta ao Gemini deve continuar valendo para o recorte da tela.
- **`geminiAsk()`** — REST `v1beta/…:generateContent`, sem SDK. Percorre `geminiModels()` em
  ordem: 429/404/5xx/resposta vazia → próximo modelo; 401/403 aborta (chave ruim, trocar de
  modelo não resolve). Devolve `{text, model}` e a UI mostra qual modelo respondeu.
- **`renderAll()` chama `chatSyncCtx()`** — a linha de contexto do painel de chat acompanha os
  filtros. Se você acrescentar um filtro novo, reflita-o em `filterSummary()` **e** em
  `chatSyncCtx()`, senão o modelo responde sobre um recorte que não está na tela.
- **Clima** — `initWeather()` só roda se houver `OPENWEATHER_API_KEY`; geolocalização negada,
  chave ausente ou erro de rede resultam em pílula oculta, nunca em erro visível.

### Núcleo

- **`key(s)`** — canonicalizador de nome de país (NFD, remove diacríticos, `&`→`and`, `st.`→`saint`,
  `rep./republic`→`rep`, descarta `the`/`of`, tira não-alfanuméricos). É a *única* chave de junção
  entre o CSV, o `GEO_REF` e o mapa. Qualquer nova conciliação de nome passa por aqui.
- **`GEO_REF`** (IIFE) — `Map` de `key(nome) → {continent, name}` sobre ~200 países canônicos, mais
  um objeto `alias` com variantes PT/EN. **`TOPO_ALIAS`** faz o mesmo para os nomes do Natural Earth
  (`'United States of America'→'USA'`, `'Dem. Rep. Congo'→'DR Congo'`, …). País que não casa
  continua em todas as estatísticas e só some do mapa — nunca descarte a linha.
- **`parseCSV` / `sniffDelim` / `num`** — fareja separador (`,` `;` tab `|`), trata BOM, aspas e
  decimal com vírgula ou ponto.
- **`buildDataset`** — casa cabeçalhos por regex PT/EN (`HEAD_RULES`) em qualquer ordem; se falhar,
  cai para heurística de coluna numérica e empilha um aviso em `ds.warnings`, exibido no banner.
- **`equalEarth` / `decodeArcs` / `buildPaths`** — projeção Equal Earth em forma fechada e decoder
  de TopoJSON escritos à mão (arcos delta, índice negativo = `~i` invertido). `buildPaths` **corta
  o subpath no antimeridiano** (salto de longitude > 180°); sem isso Rússia e Fiji desenham uma
  faixa horizontal atravessando o mapa. O bbox largo dessas geometrias é legítimo, não é o bug.
- **Estatística** — `describe` (σ amostral, n−1), `pearson`, `pValue` (z de Fisher + cauda normal
  Abramowitz & Stegun 26.2.17), `etaSquared` (variância explicada pelo continente — o substituto
  honesto para "correlação com o país", que é categórico).
- **`textOn(hex)`** — escolhe texto claro/escuro por luminância relativa. Use em toda célula com
  fundo dinâmico (matriz de correlação, chips de continente).

### Estado e render

Objeto único `APP` (`ds`, `metric`, filtros `f`, `mapScale`, `sort`, `view`…). Toda interação
muda `APP` e chama `renderAll()`, que refiltra via `filtered()` e repinta os painéis
(`renderKPIs`, `renderMap`, `renderStatTable`, `renderCorr`, `renderScatter`, `renderRank`,
`renderContinents`, `renderTable`). Não há render incremental — mantenha assim.

## Marca

Os valores do brandbook já foram extraídos (o PDF é vetorial com fontes em *subset*, sem texto
copiável) e vivem nos tokens CSS: Poppins, Admiral Blue `#003663`, Citron `#B9DA00`,
Chartreuse `#94C356`. Não reabra o PDF.

Armadilha conhecida: **citron puro não passa contraste sobre branco**. O tema claro
(`html[data-theme="light"]`) redefine `--accent:#5F7A00` para texto/borda e `--logo:var(--admiral)`;
`#B9DA00` fica só em preenchimentos. O wordmark no SVG usa `fill="var(--logo)"` — não volte a
cravar a cor.

## Editando

Edite `Dashboards/index.html` diretamente; ele é a fonte da verdade. (Ele nasceu da concatenação
de `part1.html` + `app1.js` + `app2.js` por um `build.ps1` no scratchpad da sessão, mas esses
arquivos são efêmeros e não devem ser recriados.)

Cuidado com o encoding: o arquivo é UTF-8 sem BOM e cheio de acentos. `sed` e `perl` neste
ambiente comem barras invertidas em literais de regex — `̀` já virou `0300` no
`normalize('NFD')` mais de uma vez. Para trechos com escapes, use a ferramenta Edit e confira
com `grep -n ... | cat -A`.
