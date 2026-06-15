# The Bootstrap — Makai Forger

Sistema unificado de atualização e provisionamento do **Makai Forger**. Quando o app inicia, o Bootstrap abre, baixa o `resonance.json`, compara versões, e atualiza cada componente que precisa.

## Componentes

| Componente | Versão | O que faz |
|---|---|---|
| **Bootstrap** | `v0.0.1` | Orquestrador de inicialização, atualização e provisionamento |
| **Installer API** | `v0.0.7` | Classifica e extrai instaladores de jogos (7z, innoextract, Wine) |
| **Catálogo DB** | `v0.0.1` | ~179k jogos Steam + custom com metadados, Proton, preços |
| **Proton Data DB** | `v0.0.1` | Dados de compatibilidade Proton (~81k jogos do ProtonDB) |
| **Fork Catalog Data** | `v0.0.1` | Catálogo de forks Proton/Wine/DXVK/VKD3D (30 forks, 59 releases) |
| **Game DLLs Data** | `v0.0.1` | DLLs recomendadas + winetricks por jogo (13 jogos, 10 DLLs) |
| **Releases Data** | `v0.0.1` | Metadados consolidados das releases do ecossistema |

## Como funciona

```
App inicia → Bootstrap abre →
  1. Baixa resonance.json de MakaiForger/the-bootstrap
  2. Pra cada componente:
     ├── Versão local < remota? → download (do release correspondente)
     ├── SHA256 confere? → extrai
     └── Pronto → segue
  3. Abre o app principal
```

Cada componente tem seu próprio release neste repositório, seu próprio versionamento, seu próprio asset. O `resonance.json` mapeia tudo.

---

## Bootstrap

Orquestrador de inicialização. Gerencia o ciclo de vida do app: verificação de atualizações, download de componentes, extração e inicialização da aplicação principal.

**Tecnologia:** TypeScript

**Estrutura:**

```
├── setup-window.ts             → Janela de splash/progresso
├── resource-manager.ts         → Gerenciador de downloads e versões
├── venv.ts                     → Gerenciamento do ambiente virtual
├── update-manager.ts           → Lógica de atualização
├── autoupdater/
│   ├── index.ts                → Ponto de entrada do autoupdater
│   ├── check-for-updates.ts    → Verificador de novas versões
│   └── restart-and-install-update.ts → Aplica e reinicia
└── background.png              → Identidade visual
```

---

## Installer API

API modular de classificação e extração de instaladores de jogos. Detecta automaticamente o tipo do instalador e usa extração nativa (Linux) sempre que possível, com Wine apenas como fallback.

**Tecnologia:** JavaScript (Node.js)

### Arquitetura

```
index.js               → API pública (analyze + extract + classify)
classifier.js          → Classificador de instalador (4 camadas)
utils.js               → Helpers (scanDir, scanExes, guessGameName, etc.)
overrides-loader.js    → Carrega regras fixas por jogo (override)
overrides.json         → Regras de override (arquivo único)
overrides/
  └── 00-terareforged.json
extractors/
  ├── archive.js       → 7z, rar, zip, tar, tar.gz (nativo)
  ├── exe-companions.js→ EXE pequeno + archives enormes + registry Wine
  ├── inno-std.js      → InnoSetup padrão (innoextract)
  ├── iso.js           → Imagens ISO
  ├── portable.js      → Cópia direta de pasta
  ├── sfx-nsis.js      → SFX / NSIS auto-extrável (7z)
  └── wine-fallback.js → Wine + Proton (último caso)
```

### API

**`analyze(sourcePath, gameId?, gameTitle?)`** — Classifica um instalador e retorna metadados.

```js
const { analyze } = require('./installer-api');
const info = analyze('/path/to/setup.exe', 'game-123', 'Meu Jogo');
// { type, method, needsWine, confidence, gameName, companionArchives, ... }
```

**`extract(installInfo, options)`** — Extrai um instalador com base no resultado do analyze.

```js
const { extract } = require('./installer-api');
const result = extract(info, {
  destPath: '/home/user/Games/MeuJogo/pfx/drive_c',
  protonPath: '/path/to/proton',
  source: 'catalog',
  onProgress: (msg) => console.log(msg),
});
// { success, destDir, candidates, registryNeeded, error }
```

### Tipos de instalador

| Tipo | Nativo? | Ferramenta |
|------|---------|-----------|
| pure-archive | Sim | 7z / unrar / unzip / tar |
| exe-with-companions | Sim (dados) + Wine (registry) | 7z + wine64 |
| sfx / nsis | Sim | 7z x exe |
| inno-std | Sim | innoextract |
| inno-custom | Não (Wine) | wine64 |
| portable | Sim | cp -r |
| iso | Sim | 7z x |
| unknown | Não (Wine) | wine64 |

### CLI

```bash
node installer-api/index.js --analyze setup.exe
node installer-api/index.js --extract setup.exe --dest /destino --source catalog
```

---

## Catálogo DB

Banco de dados mestre de jogos do Makai Forger. SQLite com ~179k jogos do catálogo público Steam, mais jogos custom (Genshin Impact, NIKKE, etc.).

### Schema

```sql
CREATE TABLE games (
    objectId             TEXT PRIMARY KEY,
    title                TEXT NOT NULL,
    shop                 TEXT NOT NULL,
    genres               TEXT,
    libraryImageUrl      TEXT,
    libraryHeroImageUrl  TEXT,
    iconUrl              TEXT,
    shortDescription     TEXT,
    developer            TEXT,
    publisher            TEXT,
    minimum              TEXT,
    recommended          TEXT,
    releaseYear          TEXT,
    release_date         TEXT,
    recommendedProton    TEXT,
    protonConfidence     TEXT,
    protonSource         TEXT,
    protonFallback       TEXT,
    protonAlternatives   TEXT,
    downloadSources      TEXT,
    downloads            TEXT,
    screenshots          TEXT,
    movies               TEXT,
    pcRequirements       TEXT,
    estimated_owners     INTEGER
);

CREATE VIRTUAL TABLE games_fts USING fts5(
    objectId UNINDEXED, title, genres, developer, publisher,
    content='games', content_rowid='rowid'
);
```

### Pipeline

```bash
node scripts/fetch-games.cjs              # Buscar metadados Steam
node scripts/populate-download-games.cjs   # Popular SQLite
node scripts/run-migration.cjs             # Mesclar jogos custom
gzip -kf resources/catalogo.db             # Compactar
```

---

## Proton Data DB

Banco de dados de compatibilidade Proton. ~81k jogos analisados do ProtonDB, com relatórios, notas, e recomendações de Proton por jogo.

---

## Fork Catalog Data

Banco de dados SQLite com catálogo completo de forks de Proton, Wine, DXVK e VKD3D — versões, metadados e URLs de download. Serve como cache local para evitar chamadas excessivas às APIs.

### Schema

**`fork_tools`** — definição de cada fork (30 entries)

| Coluna | Descrição |
|--------|-----------|
| `id` | Identificador único (ex: `proton-ge`) |
| `title` | Nome legível |
| `category` | proton / wine / dxvk / vkd3d |
| `endpoint` | URL da API de releases |
| `directory_name_format` | Template da pasta |
| `keywords` | JSON array de sinônimos |
| `support_latest` | 1 se suporta latest |
| `asset_position` | Posição do asset |
| `asset_filter` | Regex para filtrar assets |
| `sort_order` | Ordem de exibição |

**`fork_releases`** — cache de versões (59 releases)

---

## Game DLLs Data

Banco de dados SQLite com recomendações de DLLs e comandos winetricks para jogos rodando via Proton no Makai Forger.

### Schema

**`dll_catalog`** — DLLs conhecidas (vcrun2022, mfplat, etc.) com nível de impacto e comando winetricks.

**`game_dlls`** — mapeamento por jogo: Steam AppID ou custom, DLLs necessárias, winetricks e overrides.

Dados atuais: 13 jogos mapeados, 10 DLLs catalogadas.

### Fluxo no app

```
Usuário → "Verificar DLLs" → checkGameDlls.ts →
  ProtonRecommendationService.installGameDlls() →
  Python RPC → winetricks no prefixo
```

---

## Releases Data

Metadados consolidados de todas as releases do ecossistema. Usado como fallback e referência centralizada.

---

## Política de Releases

> **Nunca apague um release existente.**

Cada release representa um ponto imutável na história de um componente. Para corrigir ou evoluir, crie um **novo release** com versão incremental. Releases antigos continuam disponíveis para rollback e rastreabilidade.

### Versionamento

```
<nome-do-componente>-v<major>.<minor>.<patch>
```

Exemplo: `installer-api-v0.0.7`, `catalogo-db-v0.0.1`

### ⚠️ Regra obrigatória ao publicar

Sempre que você publicar uma nova versão (release) de QUALQUER recurso, você DEVE atualizar o `resonance.json` na raiz deste repositório:

1. Publique o release no GitHub (tag + asset)
2. Faça upload do asset para o GitLab (project ID correspondente)
3. **Edite o `resonance.json`**:
   - Atualize `version` (incremente)
   - Atualize `tag` para a nova tag
   - Atualize `sha256` e `size` do novo asset
4. Commit e push

### Por quê?

O Makai Forger, ao iniciar, baixa o `resonance.json` deste repositório e compara as versões locais com as versões remotas. Se o `resonance.json` não for atualizado, o aplicativo NUNCA saberá que uma nova versão existe e nunca baixará a atualização.

---

## Licença

The MIT License (MIT)

Copyright (c) 2026 ProtonFrog

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
