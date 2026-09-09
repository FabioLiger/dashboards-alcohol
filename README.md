# Dashboards Alcohol — Deixa Comigo Bebidas

Painel de análise de consumo de bebidas alcoólicas por país, desenvolvido para a
**Deixa Comigo Bebidas** (divisão Nitro). Interface e dados 100% em português do Brasil.

## O que é

Uma entrega única e autocontida: [`Dashboards/index.html`](Dashboards/index.html) — um único
arquivo HTML (~200 KB) com todo o CSS e JavaScript embutidos. Sem build, sem bundler, sem
dependência de CDN em tempo de execução (a única fonte externa é a fonte Google, com fallback
de stack).

O painel oferece:

- Mapa-múndi coroplético (projeção Equal Earth) por métrica de consumo
- KPIs agregados, ranking, tabela e matriz de correlação entre bebidas
- Agrupamento e comparação por continente
- Dispersão (scatter) entre métricas e estatísticas (correlação de Pearson, p-valor, η²)
- Tema claro/escuro com paleta da marca Nitro
- **Fale com seus DADOS** — chat com IA (Google Gemini) que responde sobre a seleção atual
- Previsão do tempo do usuário no cabeçalho (OpenWeatherMap, via geolocalização do navegador)

**Nenhum dado de consumo é hard-coded** — tudo vem do CSV importado em tempo de execução pelo
usuário. A única informação embutida no código é metadado geográfico (continentes, aliases de
nomes de país, malha de fronteiras para o mapa).

## Como usar

Abra o arquivo diretamente no navegador:

```bash
start "" "Dashboards/index.html"
```

O painel abre vazio de propósito. Arraste [`Dados/drinks.csv`](Dados/drinks.csv) (ou sua própria
planilha no mesmo formato) para a janela, ou use o botão **Importar planilha**.

> Sob `file://` o `fetch()` é bloqueado pelo navegador, então o auto-load do CSV a partir da
> pasta `Dados/` falha em silêncio (por design). Para testar esse caminho, sirva o projeto por
> HTTP a partir da raiz:
>
> ```bash
> python -m http.server 8080
> ```

### Formato esperado do CSV

```
country,beer_servings,spirit_servings,wine_servings,total_litres_of_pure_alcohol
Afghanistan,0,0,0,0.0
Albania,89,132,54,4.9
...
```

Cabeçalhos em português ou inglês são reconhecidos automaticamente; separador (`,` `;` tab `|`)
e decimal (`,` ou `.`) são detectados na importação.

## Chaves de API (`.env`)

O chat com IA e a previsão do tempo precisam de chaves. Elas ficam em um `.env` na **raiz do
projeto** — git-ignorado, lido em tempo de execução pelo painel. Não existe `config.js`.

```bash
cp .env.example .env   # depois edite com suas chaves
```

```
GEMINI_API_KEY=...                 # https://aistudio.google.com/apikey
OPENWEATHER_API_KEY=...            # https://openweathermap.org/api
GEMINI_MODELS=gemini-2.5-flash,gemini-2.5-flash-lite,gemini-2.0-flash
```

`GEMINI_MODELS` é a **cadeia de fallback**: o primeiro modelo é o preferido e, em caso de erro,
limite de cota (429) ou modelo indisponível, o painel tenta o próximo automaticamente e mostra
qual modelo respondeu.

> O `.env` só é lido quando o painel é servido por HTTP (sob `file://` o `fetch` é bloqueado).
> Nesse caso o chat pede a chave uma única vez e a guarda no `localStorage` do navegador.
> A previsão do tempo simplesmente não aparece se a chave faltar ou se a geolocalização for negada.

### O chat respeita os filtros

O contexto enviado ao Gemini é montado a partir de `filtered()` — o mesmo recorte que está na
tela. Continente, países fixados, faixa de consumo e métrica em foco entram no prompt, junto com
as estatísticas e a tabela da seleção. Mude um filtro e a próxima resposta acompanha.

## Estrutura do repositório

```
.env.example                                  modelo das chaves de API (copie para .env)
Dados/drinks.csv                              amostra de entrada
Referencias/nitro_brand_book_by_pomelli.pdf   brandbook da marca (vetorial)
Dashboards/index.html                         o produto
CLAUDE.md                                     guia de arquitetura e convenções para IA/devs
```

## Desenvolvimento

Não há testes automatizados, linter ou pipeline de build — o arquivo é a fonte da verdade e é
editado diretamente. Detalhes de arquitetura (parsing, projeção geográfica, estatística, estado
de render) e as invariantes do projeto estão documentados em [`CLAUDE.md`](CLAUDE.md).

Teste sempre nos dois temas (claro/escuro) antes de enviar alterações — regressões de contraste
no tema claro já ocorreram mais de uma vez.
