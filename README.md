# DesertPB — INSA Front

WebGIS desenvolvido pelo INSA (Instituto Nacional do Semiárido) para monitorar a
vulnerabilidade à desertificação no Semiárido da Paraíba. Construído com Vue 3
+ Vite, renderiza vector tiles servidos localmente (sem dependência de servidor
de mapas externo) e exibe camadas temáticas — índices compostos, escores de
vulnerabilidade e indicadores brutos — sobre um mapa base interativo.

A aplicação tem quatro telas: **Início** (apresentação do projeto), **Mapa**
(a aplicação principal — sidebar de camadas + Leaflet), **Painel Interativo**
(dashboard comparativo entre municípios) e **Sobre** (histórico do projeto e
equipe).

## Ambiente de homologação

[![Abrir aplicação](https://img.shields.io/badge/Abrir%20aplicação-insa--front-blue?style=for-the-badge)](https://marcellobenigno.github.io/insa-front/)

Ambiente atualizado automaticamente a cada push na branch `main`.
Use este link para visualizar e validar as camadas junto à equipe.

## Pré-requisitos

- **Node.js** `^20.19.0` ou `>=22.12.0`
- Para o pipeline de dados (GeoPackage → vector tiles): **GDAL**, **Tippecanoe** e **Python 3** — ver [Pipeline de dados](#pipeline-de-dados-geopackage--vector-tiles)

## Instalação e execução

```sh
npm install
npm run dev       # servidor de desenvolvimento com hot-reload
npm run build     # build de produção em dist/
npm run preview   # preview do build de produção
```

## Comandos disponíveis

| Comando | Descrição |
|---|---|
| `npm run dev` | Servidor de desenvolvimento (Vite) |
| `npm run build` | Build de produção → `dist/` |
| `npm run preview` | Preview do build de produção |
| `npm run lint` | oxlint + eslint com auto-fix (sequencial) |
| `npm run format` | Formata `src/` com Prettier |
| `npm run deploy:tiles` | Empacota e envia `public/tiles/` para o servidor de tiles em produção |

Não há suíte de testes configurada.

## Arquitetura

```
src/
├── components/
│   ├── MapContainer.vue          # instância do Leaflet; observa o store e nunca guarda estado de camada
│   ├── AppNavbar.vue             # header fixo — navegação entre as 4 telas + toggle de tema
│   ├── AppSidebar.vue            # painel lateral: árvore de camadas (accordion recursivo)
│   ├── SidebarTreeGroup.vue      # renderiza recursivamente um nó de grupo da árvore de camadas
│   ├── LayerCard.vue             # card por camada folha (visibilidade, opacidade, legenda, busca)
│   ├── GeoSearch.vue             # busca geocodificada + coordenadas (DD, DMS, endereço)
│   ├── CoordDisplay.vue          # overlay de coordenadas do cursor em tempo real (DD e DMS)
│   ├── HeroCarousel.vue          # carrossel de fotos do semiárido, fundo do hero da Início
│   ├── WebGisTour.vue            # tour guiado de onboarding (primeira visita)
│   ├── LayerChartModal.vue       # gráfico de distribuição por classe (por camada)
│   ├── DashboardMiniMap.vue      # mini-mapa Leaflet independente do dashboard
│   ├── DashboardTable.vue        # tabela ordenável de municípios no dashboard
│   ├── DashboardChart.vue        # gráfico de barras do dashboard
│   └── DashboardPieChart.vue     # gráfico de pizza (distribuição por classe) do dashboard
├── composables/
│   ├── useSidebar.js             # estado do accordion e colapso da sidebar (singleton)
│   └── useTheme.js               # tema claro/escuro, persistido em localStorage (singleton)
├── stores/
│   └── mapStore.js               # Pinia — camada base ativa, visibilidade/opacidade dos overlays
├── config/
│   └── layers.js                 # ÚNICA fonte de verdade das camadas (OVERLAY_TREE)
├── utils/
│   ├── mapRenderer.js            # pinta feições no canvas a partir de styles.json
│   ├── mapPopup.js               # monta o HTML do popup de clique no mapa
│   └── createDashboardMvtLayer.js # camada MVT simplificada, exclusiva do mini-mapa do dashboard
├── router/
│   └── index.js                  # Vue Router — 4 rotas, hash history
├── views/
│   ├── InicioView.vue            # landing page (/)
│   ├── HomeView.vue              # mapa principal (/mapa)
│   ├── DashboardView.vue         # painel comparativo entre municípios (/dashboard)
│   └── SobreView.vue             # sobre o projeto e equipe (/sobre)
└── assets/
    ├── styles.json                # estilos de legenda, gerado por scripts/styles.py
    ├── stats.json                 # área por classe, gerado por scripts/stats.py
    ├── dashboard_stats.json       # cruzamento índices × municípios, gerado por scripts/dashboard_stats.py
    └── search_index.json          # índice de busca por atributo, gerado por scripts/search_index.py
```

**Fluxo de dados das camadas (tela Mapa):**

1. `config/layers.js` define `OVERLAY_TREE` — a única fonte de verdade das camadas.
2. `mapStore.js` (Pinia) mantém o estado reativo: camada base ativa e visibilidade/opacidade de cada overlay.
3. `AppSidebar.vue` / `SidebarTreeGroup.vue` / `LayerCard.vue` leem e mutam o store, renderizando `OVERLAY_TREE` como uma árvore recursiva de qualquer profundidade.
4. `MapContainer.vue` observa o store via watchers e aplica as mudanças ao mapa via Leaflet — nunca guarda estado de camada próprio.

O painel comparativo (`/dashboard`) é um fluxo **independente**: consome `dashboard_stats.json` e `stats.json` diretamente, sem passar por `OVERLAY_TREE`/`mapStore.js`, e mantém sua própria instância de mapa Leaflet em `DashboardMiniMap.vue`.

> **Restrição importante:** Leaflet não funciona em SSR. Cada instância do mapa é inicializada em `onMounted` e destruída em `onUnmounted`. Nunca acesse `L` nem a instância do mapa fora desses hooks.

---

## Funcionalidades da interface

### Sidebar (tela Mapa)

- **Árvore hierárquica de camadas** — `OVERLAY_TREE` é renderizado recursivamente (não um accordion de 2 níveis fixo): Limites, IVD, Índices de Vulnerabilidade (IVS/IVV/IVC/IVM → seus Escores) e Indicadores de Vulnerabilidade (dados brutos por trás dos escores), cada grupo/subgrupo expansível.
- **Filtro de camadas por nome** — campo de texto acima das categorias filtra em tempo real (case-insensitive).
- **Badge de visibilidade** — indicador numérico por categoria mostra quantas camadas estão ativas.
- **Colapso da sidebar** — libera espaço pro mapa; estado e accordion vivem em `useSidebar()` (singleton em nível de módulo).

### Painel de busca por atributo (por camada)

Acessado pelo ícone de lupa em cada `LayerCard`:

- **Campos string** — busca por substring, case-insensitive
- **Campos numéricos** — suporta operadores `=`, `>`, `>=`, `<`, `<=`
- **Destaque visual no mapa** — feições que batem mantêm a cor temática; as demais ficam acinzentadas (lido tile a tile, conforme carregam)
- **Contagem de resultados** — lida de `src/assets/search_index.json` (todos os atributos do GeoPackage, sem geometria), não dos tiles renderizados — garante que a contagem não dependa do que está na viewport atual

### GeoSearch (rodapé da sidebar)

- Busca por endereço via Nominatim (geocodificação)
- Entrada de coordenadas em DD (decimal) ou DMS (graus, minutos, segundos)
- Resultados restritos ao bounding box do Semiárido da PB

### CoordDisplay

Overlay no canto inferior do mapa exibe as coordenadas do cursor em DD e DMS em tempo real.

### Painel Interativo (`/dashboard`)

Cruza os 5 índices compostos (IVS, IVV, IVC, IVM, IVD) com os municípios do Semiárido PB: tabela ordenável, gráfico de barras, gráfico de pizza (distribuição por classe) e mini-mapa coroplético — selecionar uma linha na tabela destaca e centraliza o município correspondente no mini-mapa.

### Tema claro/escuro

Alternado pelo botão em `AppNavbar.vue`, disponível em todas as telas. Persistido em `localStorage` (`insa-theme`). `--accent` é o verde da marca DesertPB.

### Tour guiado

`WebGisTour.vue` — tour de onboarding apontando para elementos reais da interface, exibido na primeira visita (`localStorage`, chave `insa-tour-completed`).

---

## Referência: `src/config/layers.js`

Este é o **único arquivo que você precisa editar** para controlar quais camadas existem, como elas aparecem na sidebar e o que o popup de clique exibe. Não há nenhuma outra configuração de camadas espalhada pela aplicação.

### Estrutura geral

| Export | Descrição |
|---|---|
| `BASE_LAYERS` | Mapas de fundo (Google Satellite ★, Streets, Hybrid, Terrain, OSM, OSM Dark) — selecionados via radio button. ★ = ativo por padrão. |
| `OVERLAY_TREE` | Lista de **nós recursivos** — cada nó é um grupo (`layer: null` + `children`) ou uma folha (`layer: {...}`, sem `children`). A ordem de cada array `children` é a ordem exibida na sidebar. |
| `OVERLAY_LAYERS` | Gerado automaticamente por uma varredura recursiva de `OVERLAY_TREE` — **nunca edite diretamente**. |

Se uma camada é ao mesmo tempo um índice composto e "pai" de outras camadas (ex. IVS tem escores abaixo dela), modele como um grupo cujo **primeiro filho** é a folha da camada composta — nunca coloque `layer` e `children` no mesmo nó.

### Campos de uma camada folha

```js
layer: {
  // ── Obrigatórios ────────────────────────────────────────────────────────
  label:        'Rótulo exibido no menu e no popup',
  meta:         'Descrição curta exibida abaixo do rótulo na sidebar',
  url:          VECTOR_TILES_URL,        // aponta para public/tiles/ — não alterar
  sourceLayer:  'nome_exato_no_gpkg',    // deve bater byte-a-byte com o nome no GeoPackage
  zIndex:       20,                      // maior = fica acima de outras camadas no mapa
  active:       false,                   // true = camada visível ao carregar a página

  // ── Barra de busca ──────────────────────────────────────────────────────
  searchFields: ['campo1', 'campo2'],    // campos pesquisáveis
  fieldTypes:   { campo1: 'string',      // 'string' (substring) ou 'number' (operadores)
                  campo2: 'number' },

  // ── Popup de clique ─────────────────────────────────────────────────────
  popUpFields:  ['campo1'],              // campos exibidos, na ordem declarada
                                          // se omitido: mostra todos os campos exceto id/gid/fid
  descFields:   { campo1: 'Rótulo amigável' }, // se ausente, usa o nome técnico do campo
  noPopup:      true,                    // omitir ou false = popup normal;
                                          // true = camada excluída das buscas por clique
                                          // (ex. limites, sem atributo relevante)
  renderAs:     'geojson',               // só necessário para camadas fora do padrão MVT
                                          // (hoje, só focos_queimadas — ver seção abaixo)
}
```

`descFields` deve conter rótulos legíveis em português, nunca o nome técnico do campo como valor (ex. `{ ivd: 'IVD' }` está errado; `{ ivd: 'Índice de Vulnerabilidade à Desertificação' }` está certo) — esses rótulos aparecem no popup e no cabeçalho do painel de busca ("Buscar por `<rótulo>`").

**Convenção de `zIndex`:**

| Faixa | Uso |
|---|---|
| 1 | Camadas base |
| 10–28 | Índices compostos (IVD, IVS, IVV, IVC, IVM) e seus Escores de Vulnerabilidade |
| 30–43 | Indicadores de Vulnerabilidade (dados brutos por trás dos Escores) |
| 50+ | Limites administrativos (sempre por cima) |

**Exceção arquitetural — `focos_queimadas`:** é a única camada que não usa vector tiles MVT. Sem atributos e com apenas 511 feições, é servida como um único GeoJSON estático (`renderAs: 'geojson'`) e renderizada em `L.circleMarker`, que cai por padrão no `markerPane` do Leaflet — por isso fica sempre acima de todas as outras camadas, sem precisar de `zIndex` especial.

---

## Pipeline de dados (GeoPackage → Vector Tiles)

> ⚠️ **Sempre que uma camada for adicionada, removida ou alterada no GeoPackage, todos os passos abaixo devem ser refeitos do zero.** Não existe atualização parcial — cada `.pbf` em `public/tiles/` contém todas as camadas daquele tile, empacotadas juntas pelo Tippecanoe.

### Ferramentas necessárias

```bash
brew install gdal tippecanoe python3
```

Todos os comandos rodam a partir da **raiz do projeto**; os scripts Python ficam em `scripts/`.

### Passo 1 — Exportar do GeoPackage para GeoJSON

```bash
# Listar camadas disponíveis
ogrinfo -q data/dados_insa.gpkg

# Exportar uma camada
ogr2ogr -f GeoJSON data/geojson/<camada>.geojson \
  data/dados_insa.gpkg <camada> \
  -t_srs EPSG:4326
```

> `focos_queimadas` é a exceção — não passa por este passo nem pelo Tippecanoe, ver "Exceção arquitetural" acima.

### Passo 2 — Gerar o `.mbtiles` com Tippecanoe

```bash
tippecanoe \
  -o data/mbtiles/insa_layers.mbtiles \
  -z14 -Z2 \
  --no-feature-limit \
  --no-tile-size-limit \
  --extend-zooms-if-still-dropping \
  --no-tile-compression \
  --no-tiny-polygon-reduction \
  --force \
  data/geojson/*.geojson
```

| Flag | Motivo |
|---|---|
| `-z14 -Z2` | Gera tiles do zoom 2 (visão geral) ao 14 (detalhe) |
| `--no-feature-limit` / `--no-tile-size-limit` | Não descarta feições por limite de quantidade/tamanho por tile |
| `--extend-zooms-if-still-dropping` | Aumenta zoom máximo se ainda estiver descartando dados |
| `--no-tile-compression` | Salva `.pbf` sem compressão (necessário para leitura direta pelo browser) |
| `--no-tiny-polygon-reduction` | **Obrigatório.** Sem essa flag, polígonos pequenos são fundidos/descartados nos zooms baixos por padrão — já causou um bug real de busca (uma feição existia no GeoPackage mas sumia do tile, retornando "Nenhum resultado encontrado" para um filtro que na verdade tinha match) |
| `--force` | Sobrescreve o `.mbtiles` existente sem perguntar |

O `.mbtiles` gerado (~200 MB) é gitignored — não commitar.

### Passo 3 — Apagar tiles antigos e reextrair

```bash
rm -rf public/tiles/insa_layers      # OBRIGATÓRIO — nunca pular
python scripts/export.py             # grava public/tiles/insa_layers/{z}/{x}/{y}.pbf

# focos_queimadas precisa ser regerado por último (o rm -rf acima apaga junto):
ogr2ogr -f GeoJSON public/tiles/insa_layers/focos_queimadas.geojson \
  data/dados_insa.gpkg focos_queimadas -t_srs EPSG:4326
```

### Passo 4 — Extrair estilos

```bash
python scripts/styles.py    # gera src/assets/styles.json a partir do layer_styles (QML) do GeoPackage
```

> ⚠️ **Sobrescreve `styles.json` inteiro.** Camadas `singleSymbol`/stroke-only (limites, `focos_queimadas`) não são capturadas automaticamente — precisam de entrada manual. Ver a seção correspondente no `CLAUDE.md` para o JSON exato a restaurar após rodar este passo.

### Passo 5 — Gerar estatísticas de área

```bash
python scripts/stats.py    # gera src/assets/stats.json
```

### Passo 6 — Gerar estatísticas do dashboard

```bash
python scripts/dashboard_stats.py    # gera src/assets/dashboard_stats.json
```

Só precisa rodar de novo se o estilo de um dos 5 índices compostos (IVS/IVV/IVC/IVM/IVD) mudar.

### Passo 7 — Gerar o índice de busca

```bash
python scripts/search_index.py    # gera src/assets/search_index.json
```

Lido direto do GeoPackage via SQLite — não deriva de nenhum outro `.json` gerado, então precisa rodar sempre que o GeoPackage mudar (nada mais no pipeline avisa que ficou desatualizado).

---

## Deploy dos tiles para produção

Após regenerar os tiles localmente (Passos 2–3 acima):

```bash
npm run deploy:tiles
```

**Script:** `scripts/deploy-tiles.sh`
**Servidor:** `ubuntu@2.25.137.181` (`sistema.sigrural.com.br`) — acesso via chave SSH, sem senha
**Caminho remoto:** `/var/www/html/tiles/insa_layers/`

O script compacta `public/tiles/insa_layers/` (~42 MB), envia via SCP, substitui os tiles antigos no servidor e limpa os arquivos temporários.

## Deploy da aplicação (GitHub Pages)

`.github/workflows/deploy.yml` builda e publica automaticamente a cada push na `main` (`npm run build` → `dist/` → GitHub Pages). O build de produção lê `.env.production` (`VITE_TILES_URL`), então **os tiles precisam estar publicados no servidor (`npm run deploy:tiles`) antes ou junto do push do código** — caso contrário o site em produção aponta para tiles que ainda não existem.

---

## Como adicionar uma nova camada

### Dados

- [ ] Adicione a camada ao `dados_insa.gpkg` no QGIS e salve o estilo
- [ ] Exporte para GeoJSON (Passo 1)
- [ ] Regere o `.mbtiles` (Passo 2 — o glob `data/geojson/*.geojson` já pega o novo arquivo automaticamente)
- [ ] Regere os tiles (Passo 3)
- [ ] Regere os estilos (Passo 4) e restaure as entradas manuais em `styles.json`
- [ ] Se stroke-only, adicione a entrada manualmente em `styles.json`
- [ ] Regere as estatísticas (Passo 5)
- [ ] Se for um dos 5 índices compostos, regere as estatísticas do dashboard (Passo 6)
- [ ] Regere o índice de busca (Passo 7)

### Código

Adicione um nó em `OVERLAY_TREE`, dentro de `src/config/layers.js`, na categoria correta — ver [Referência: `src/config/layers.js`](#referência-srcconfiglayersjs) para a descrição completa de cada campo.

> `sourceLayer` deve ser **idêntico** ao nome da camada no GeoPackage — qualquer divergência faz os tiles não renderizarem silenciosamente.

---

## Estrutura de `data/`

```
data/
├── dados_insa.gpkg          # fonte primária — GeoPackage com todas as camadas e estilos
├── geojson/                 # camadas exportadas em GeoJSON (intermediário, gitignored)
├── mbtiles/
│   └── insa_layers.mbtiles  # vector tiles empacotados (~200 MB, gitignored)
```

`public/tiles/insa_layers/` (tiles finais + `focos_queimadas.geojson`) também é gitignored — regenerado a cada rodada do pipeline.

---

## Stack

| Camada | Lib |
|---|---|
| UI | Vue 3 — Composition API, `<script setup>`, JavaScript (não TypeScript) |
| Estado | Pinia (`src/stores/mapStore.js`) |
| Mapa | Leaflet 1.x |
| Vector tiles | `L.GridLayer` customizado (`MapContainer.vue`) — decodifica `.pbf` com `vector-tile` + `pbf`, pinta em `<canvas>` |
| Layout | Bootstrap 5 |
| Ícones | Bootstrap Icons — via CDN em `index.html` (não é dependência npm) |
| Gráficos | Chart.js 4.x — importado modularmente por componente, nunca registrado globalmente |
| Roteamento | vue-router 4.x — `createWebHashHistory` (GitHub Pages não faz rewrite de servidor para SPA) |
| Lint/format | Oxlint + ESLint + Prettier |

## IDE recomendada

[VS Code](https://code.visualstudio.com/) com a extensão [Vue (Official)](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (desative o Vetur se estiver instalado).
