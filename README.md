# The Bootstrap — Makai Forge

:brazil: **Português** | :us: [English](#english) | :es: [Español](#espanol)

---

<a name="portugues"></a>

# :brazil: The Bootstrap — Makai Forge

## O que é

The Bootstrap é o sistema de inicialização, atualização e provisionamento do **Makai Forge**. É a primeira coisa que roda quando o aplicativo abre. Ele garante que todos os componentes (catálogo de jogos, API de instaladores, dados Proton, forks, DLLs) estejam na versão correta antes do app principal iniciar.

---

## Como o Bootstrap gerencia versões

O Bootstrap usa **dois arquivos de metadados** para decidir o que baixar:

### 1. `resonance.json` (remoto — no GitHub)

Fica na raiz deste repositório. É o manifesto oficial com as versões mais recentes de cada componente.

```json
{
  "version": 1,
  "components": {
    "catalogo-db": {
      "version": 1,
      "tag": "catalogo-db-v0.0.1",
      "sha256": "af72a2fd...",
      "size": 55951414,
      "archive": "catalogo.db.gz"
    }
  }
}
```

### 2. `metadata.json` (local — no disco do usuário)

Fica na pasta de dados do Makai Forge no computador do usuário. Registra quais versões estão instaladas localmente.

```json
{
  "catalogo-db": {
    "version": 0,
    "path": "/home/user/.config/makaiforger/resources/catalogo.db"
  }
}
```

---

## Fluxo completo de inicialização

### Primeira execução (nunca baixou nada)

```
1. Usuário abre o Makai Forge
   ↓
2. Bootstrap abre a janela de splash
   ↓
3. Bootstrap baixa resonance.json de:
   ├── Primário: https://github.com/MakaiForge/the-bootstrap/raw/main/resonance.json
   └── Fallback: GitLab (project ID correspondente)
   ↓
4. Bootstrap procura metadata.json local
   └── Não existe (primeira vez) → considera versão local = 0 pra tudo
   ↓
5. Para cada componente em resonance.json:
   ├── versão local (0) < versão remota (1)?
   │   └── SIM → precisa baixar
   ├── 5.1 Baixa o archive do release do componente
   │   ├── Ex: https://github.com/MakaiForge/the-bootstrap/releases/download/catalogo-db-v0.0.1/catalogo.db.gz
   │   └── Salva em: ~/.cache/makaiforger/downloads/catalogo.db.gz
   ├── 5.2 Calcula SHA-256 do arquivo baixado
   ├── 5.3 Compara com o SHA-256 do resonance.json
   │   ├── Bate? → continua
   │   └── Não bate? → apaga e baixa de novo
   ├── 5.4 Extrai o archive
   │   ├── .tar.gz → extrai em ~/.config/makaiforger/resources/<componente>/
   │   └── .db.gz   → descomprime em ~/.config/makaiforger/resources/<arquivo>.db
   ├── 5.5 Cria/atualiza metadata.json local:
   │   └── "catalogo-db": { "version": 1, "path": "..." }
   └── 5.6 Passa pro próximo componente
   ↓
6. Todos os componentes atualizados
   ↓
7. Bootstrap fecha a janela de splash
   ↓
8. App principal abre
```

### Execuções seguintes (já tem algo baixado)

```
1. Usuário abre o Makai Forge
   ↓
2. Bootstrap abre splash
   ↓
3. Bootstrap baixa resonance.json (remoto)
   ↓
4. Bootstrap lê metadata.json (local)
   ↓
5. Para cada componente:
   ├── catalogo-db: local version=1, remota version=1
   │   └── IGUAL → pula (já atualizado)
   ├── installer-api: local version=8, remota version=8
   │   └── IGUAL → pula
   ├── proton-data-db: local version=1, remota version=2
   │   └── LOCAL < REMOTA → precisa atualizar!
   │       ├── Baixa proton_data.db.gz da release proton-data-db-v0.0.2
   │       ├── Verifica SHA-256
   │       ├── Extrai
   │       └── metadata.json: proton-data-db.version = 2
   └── ...
   ↓
6. Bootstrap fecha splash
   ↓
7. App principal abre
```

### Fluxo visual simplificado

```
┌──────────────┐     ┌──────────────────┐     ┌───────────────────┐
│  Usuário     │     │  GitHub          │     │  Disco Local      │
│  abre app    │────>│  resonance.json  │────>│  metadata.json    │
└──────────────┘     └──────────────────┘     └───────────────────┘
                                                     │
                                                     ▼
                                            ┌──────────────────┐
                                            │  Compara versões  │
                                            │  local vs remota  │
                                            └──────────────────┘
                                                     │
                                      ┌──────────────┴──────────────┐
                                      ▼                             ▼
                               ┌─────────────┐             ┌──────────────┐
                               │  Versões    │             │  Versões     │
                               │  iguais     │             │  diferentes  │
                               │  → pula     │             │  → baixa     │
                               └─────────────┘             └──────┬───────┘
                                                                  │
                                                                  ▼
                                                         ┌────────────────┐
                                                         │  Download do   │
                                                         │  archive do    │
                                                         │  release       │
                                                         └────────┬───────┘
                                                                  │
                                                                  ▼
                                                         ┌────────────────┐
                                                         │  Verifica      │
                                                         │  SHA-256       │
                                                         └────────┬───────┘
                                                                  │
                                                                  ▼
                                                         ┌────────────────┐
                                                         │  Extrai e      │
                                                         │  atualiza      │
                                                         │  metadata      │
                                                         └────────────────┘
```

---

## O que acontece depois de uma atualização

Quando o Bootstrap termina de baixar e extrair tudo:

1. O arquivo `metadata.json` local é atualizado com as novas versões
2. O `resonance.json` baixado também é mantido em cache (para referência)
3. Na **próxima inicialização**:
   - O Bootstrap baixa o `resonance.json` novamente
   - Lê o `metadata.json` local
   - Compara: se as versões forem iguais, **não baixa nada**
   - Só baixa de novo quando o `resonance.json` remoto tiver uma versão maior

### Exemplo prático

| Data | Ação | catalogo-db (local) | catalogo-db (remoto) | Baixou? |
|------|--------|--------------------|---------------------|-------------|
| 01/01 | Primeira execução | 0 | 1 | Sim |
| 02/01 | Abre de novo | 1 | 1 | Não |
| 05/01 | Dev publica v0.0.2 | 1 | 2 | Sim |
| 06/01 | Abre de novo | 2 | 2 | Não |
| 10/01 | Dev publica v0.0.3 | 2 | 3 | Sim |

### Auto-update: desativar atualizações automáticas

Por padrão, o Bootstrap baixa e atualiza tudo automaticamente. Para desativar esse comportamento:

1. Crie o arquivo `bootstrap.json` na pasta de dados do Makai Forge:
   ```
   ~/.config/makaiforger/bootstrap.json
   ```
2. Com o conteúdo:
   ```json
   { "auto_update": 0 }
   ```
3. Com `1` (padrão) = atualiza automaticamente. Com `0` = nunca atualiza, usa os arquivos locais como estão.

Isso é útil para:
- Usuários que querem congelar uma versão específica que funciona bem
- Desenvolvedores modificando arquivos manualmente sem o Bootstrap sobrescrever
- Testes sem interferência do sistema de atualização

---

## Componentes

| Componente | Release | Formato | Descrição |
|---|---|---|---|
| [Bootstrap](./bootstrap/) | `bootstrap-v0.0.1` | `.tar.gz` | Orquestrador de inicialização, atualização e provisionamento |
| [Installer API](./installer-api/) | `installer-api-v0.0.7` | `.tar.gz` | API de classificação e extração de instaladores |
| [Catálogo DB](./catalogo-db/) | `catalogo-db-v0.0.1` | `.db.gz` | ~179k jogos Steam + custom |
| [Proton Data DB](./proton-data-db/) | `proton-data-db-v0.0.1` | `.db.gz` | ~81k relatórios de compatibilidade Proton |
| [Fork Catalog Data](./fork-catalog-data/) | `fork-catalog-data-v0.0.1` | `.db.gz` | 30 forks, 59 releases de Proton/Wine/DXVK/VKD3D |
| [Game DLLs Data](./game-dlls-data/) | `game-dlls-data-v0.0.1` | `.db.gz` | DLLs e winetricks para 13 jogos |
| [Releases Data](./releases-data/) | `releases-data-v0.0.1` | `.tar.gz` | Metadados consolidados de fallback |

---

## Sistema de Releases

### Formato da tag

```
<nome-do-componente>-v<major>.<minor>.<patch>
```

Exemplos: `catalogo-db-v0.0.1`, `installer-api-v0.0.7`

### Política: nunca apague releases

Cada release é imutável. Se uma versão precisa de correção, crie uma nova com versão incremental. Releases antigos continuam disponíveis para rollback.

### Como publicar uma nova versão

1. Prepare o archive:
   - Código: `tar -czf <componente>.tar.gz <arquivos>`
   - Banco: `gzip -kf <arquivo>.db`

2. Crie a release no GitHub:
   - Tag: `<componente>-v<nova-versão>`
   - Asset: o archive
   - Body: descreva as mudanças

3. Faça upload no GitLab (project ID correspondente)

4. Atualize o `resonance.json` neste repositório:
   - Incremente `version` do componente
   - Atualize `tag`, `sha256` (compute com `sha256sum`), `size`
   - Commit e push

> **Atenção**: se o `resonance.json` não for atualizado, o aplicativo nunca saberá que existe uma versão nova. O Bootstrap só compara o que está no `resonance.json`.

---

## Para desenvolvedores

### Quero contribuir

Cada componente tem sua pasta com README próprio explicando schema, API, pipeline e instruções de publicação:

- [`bootstrap/`](./bootstrap/)
- [`installer-api/`](./installer-api/)
- [`catalogo-db/`](./catalogo-db/)
- [`proton-data-db/`](./proton-data-db/)
- [`fork-catalog-data/`](./fork-catalog-data/)
- [`game-dlls-data/`](./game-dlls-data/)
- [`releases-data/`](./releases-data/)

### Quero reportar um problema

Abra uma issue com:
- Componente afetado
- Versão atual (tag do release)
- Comportamento esperado
- Comportamento observado

### Quero solicitar uma funcionalidade

Abra uma issue com prefixo `[FEATURE]` no título.

---

## 📓 Histórico de atualizações

> **Regra fundamental:** nenhuma versão é jamais deletada ou sobrescrita. Cada release novo apenas adiciona uma entrada nova no histórico. Versões antigas continuam acessíveis para rollback.

### Como atualizar um componente

1. **Prepare o archive** do componente com as alterações desejadas
2. **Calcule o SHA-256:**
   ```bash
   sha256sum <arquivo>.tar.gz   # para código
   sha256sum <arquivo>.db.gz    # para banco de dados
   ```
   Copie o hash gerado (ex: `d74705009628e83737953691c5a7814212fa939673b2564be1473db8588a1fe2`)
3. **Crie o release** no GitHub:
   - Tag: `<componente>-v<nova-versão>` (ex: `bootstrap-v0.0.2`)
   - Asset: o archive
   - Body: descreva o que mudou
4. **Atualize o `resonance.json`** na raiz deste repositório:
   - Incremente o `version` do componente
   - Atualize `tag`, `sha256` (o hash do passo 2), `size` (bytes)
5. **Commit e push**

### Registro de versões

| Versão | Data | O que mudou |
|--------|------|-------------|
| `bootstrap-v0.0.1` | 2026-06-15 | Release inicial do ecossistema unificado. Inclui: Bootstrap (orquestrador), Installer API v0.0.7, Catálogo DB, Proton Data DB, Fork Catalog, Game DLLs, Releases Data. |
| `installer-api-v0.0.7` | 2026-06-15 | API de classificação e extração com 7 extractors (archive, exe-companions, inno-std, iso, portable, sfx-nsis, wine-fallback). |
| `catalogo-db-v0.0.1` | 2026-06-15 | Banco SQLite com ~179k jogos Steam + custom, schema com FTS5, metadados completos. |
| `proton-data-db-v0.0.1` | 2026-06-15 | ~81k relatórios de compatibilidade Proton do ProtonDB. |
| `fork-catalog-data-v0.0.1` | 2026-06-15 | 30 forks catalogados (Proton, Wine, DXVK, VKD3D), 59 releases em cache. |
| `game-dlls-data-v0.0.1` | 2026-06-15 | 13 jogos mapeados, 10 DLLs catalogadas com winetricks e overrides. |
| `releases-data-v0.0.1` | 2026-06-15 | Metadados consolidados de fallback para todo o ecossistema. |

### Próximas atualizações

Quando uma nova versão for publicada:
- A versão antiga **continua existindo** como release no GitHub
- O `resonance.json` é atualizado com a nova entrada
- O Bootstrap compara versões na próxima inicialização e baixa a nova se necessário
- Usuários que quiserem permanecer na versão antiga podem desativar o auto-update (`bootstrap.json` → `auto_update: 0`)

---

<a name="english"></a>

# :us: The Bootstrap — Makai Forge

## What it is

The Bootstrap is the startup, update and provisioning system of **Makai Forge**. It is the first thing that runs when the application opens. It ensures that all components (game catalog, installer API, Proton data, forks, DLLs) are at the correct version before the main app starts.

---

## How Bootstrap manages versions

Bootstrap uses **two metadata files** to decide what to download:

### 1. `resonance.json` (remote — on GitHub)

Located at the root of this repository. It is the official manifest with the latest versions of each component.

```json
{
  "version": 1,
  "components": {
    "catalogo-db": {
      "version": 1,
      "tag": "catalogo-db-v0.0.1",
      "sha256": "af72a2fd...",
      "size": 55951414,
      "archive": "catalogo.db.gz"
    }
  }
}
```

### 2. `metadata.json` (local — on the user's disk)

Stored in the Makai Forge data folder on the user's machine. It records which versions are installed locally.

```json
{
  "catalogo-db": {
    "version": 0,
    "path": "/home/user/.config/makaiforger/resources/catalogo.db"
  }
}
```

---

## Full startup flow

### First run (nothing downloaded yet)

```
1. User opens Makai Forge
   ↓
2. Bootstrap opens splash window
   ↓
3. Bootstrap downloads resonance.json from:
   ├── Primary: https://github.com/MakaiForge/the-bootstrap/raw/main/resonance.json
   └── Fallback: GitLab (corresponding project ID)
   ↓
4. Bootstrap looks for local metadata.json
   └── Doesn't exist (first time) → assumes local version = 0 for everything
   ↓
5. For each component in resonance.json:
   ├── local version (0) < remote version (1)?
   │   └── YES → needs download
   ├── 5.1 Downloads the archive from the component's release
   │   ├── Example: https://github.com/MakaiForge/the-bootstrap/releases/download/catalogo-db-v0.0.1/catalogo.db.gz
   │   └── Saves to: ~/.cache/makaiforger/downloads/catalogo.db.gz
   ├── 5.2 Calculates SHA-256 of downloaded file
   ├── 5.3 Compares with SHA-256 from resonance.json
   │   ├── Match? → continues
   │   └── No match? → deletes and downloads again
   ├── 5.4 Extracts the archive
   │   ├── .tar.gz → extracts to ~/.config/makaiforger/resources/<component>/
   │   └── .db.gz   → decompresses to ~/.config/makaiforger/resources/<file>.db
   ├── 5.5 Creates/updates local metadata.json:
   │   └── "catalogo-db": { "version": 1, "path": "..." }
   └── 5.6 Moves to next component
   ↓
6. All components updated
   ↓
7. Bootstrap closes splash window
   ↓
8. Main app opens
```

### Subsequent runs (already has data)

```
1. User opens Makai Forge
   ↓
2. Bootstrap opens splash
   ↓
3. Bootstrap downloads resonance.json (remote)
   ↓
4. Bootstrap reads metadata.json (local)
   ↓
5. For each component:
   ├── catalogo-db: local version=1, remote version=1
   │   └── EQUAL → skip (already up to date)
   ├── installer-api: local version=8, remote version=8
   │   └── EQUAL → skip
   ├── proton-data-db: local version=1, remote version=2
   │   └── LOCAL < REMOTE → needs update!
   │       ├── Downloads proton_data.db.gz from release proton-data-db-v0.0.2
   │       ├── Verifies SHA-256
   │       ├── Extracts
   │       └── metadata.json: proton-data-db.version = 2
   └── ...
   ↓
6. Bootstrap closes splash
   ↓
7. Main app opens
```

### Simplified visual flow

```
┌──────────────┐     ┌──────────────────┐     ┌───────────────────┐
│  User        │     │  GitHub          │     │  Local Disk       │
│  opens app   │────>│  resonance.json  │────>│  metadata.json    │
└──────────────┘     └──────────────────┘     └───────────────────┘
                                                     │
                                                     ▼
                                            ┌──────────────────┐
                                            │  Compare versions │
                                            │  local vs remote  │
                                            └──────────────────┘
                                                     │
                                      ┌──────────────┴──────────────┐
                                      ▼                             ▼
                               ┌─────────────┐             ┌──────────────┐
                               │  Versions   │             │  Versions    │
                               │  equal      │             │  different   │
                               │  → skip     │             │  → download  │
                               └─────────────┘             └──────┬───────┘
                                                                  │
                                                                  ▼
                                                         ┌────────────────┐
                                                         │  Download      │
                                                         │  archive from  │
                                                         │  release       │
                                                         └────────┬───────┘
                                                                  │
                                                                  ▼
                                                         ┌────────────────┐
                                                         │  Verify        │
                                                         │  SHA-256       │
                                                         └────────┬───────┘
                                                                  │
                                                                  ▼
                                                         ┌────────────────┐
                                                         │  Extract and   │
                                                         │  update local  │
                                                         │  metadata      │
                                                         └────────────────┘
```

---

## What happens after an update

When Bootstrap finishes downloading and extracting everything:

1. The local `metadata.json` is updated with the new versions
2. The downloaded `resonance.json` is also cached for reference
3. On the **next startup**:
   - Bootstrap downloads `resonance.json` again
   - Reads local `metadata.json`
   - Compares: if versions match, **nothing is downloaded**
   - Only downloads when the remote `resonance.json` has a higher version

### Practical example

| Date | Action | catalogo-db (local) | catalogo-db (remote) | Downloaded? |
|------|--------|--------------------|---------------------|-------------|
| Jan 1 | First run | 0 | 1 | Yes |
| Jan 2 | Opens again | 1 | 1 | No |
| Jan 5 | Dev publishes v0.0.2 | 1 | 2 | Yes |
| Jan 6 | Opens again | 2 | 2 | No |
| Jan 10 | Dev publishes v0.0.3 | 2 | 3 | Yes |

### Auto-update: disabling automatic updates

By default, Bootstrap downloads and updates everything automatically. To disable this behavior:

1. Create the file `bootstrap.json` in the Makai Forge data folder:
   ```
   ~/.config/makaiforger/bootstrap.json
   ```
2. With the content:
   ```json
   { "auto_update": 0 }
   ```
3. `1` (default) = auto-update enabled. `0` = never update, use local files as-is.

This is useful for:
- Users who want to freeze a specific version that works well
- Developers modifying files manually without Bootstrap overwriting them
- Testing without interference from the updater

---

## Components

| Component | Release | Format | Description |
|---|---|---|---|
| [Bootstrap](./bootstrap/) | `bootstrap-v0.0.1` | `.tar.gz` | Startup orchestrator, update and provisioning |
| [Installer API](./installer-api/) | `installer-api-v0.0.7` | `.tar.gz` | Game installer classification and extraction API |
| [Catálogo DB](./catalogo-db/) | `catalogo-db-v0.0.1` | `.db.gz` | ~179k Steam + custom games catalog |
| [Proton Data DB](./proton-data-db/) | `proton-data-db-v0.0.1` | `.db.gz` | ~81k Proton compatibility reports |
| [Fork Catalog Data](./fork-catalog-data/) | `fork-catalog-data-v0.0.1` | `.db.gz` | 30 forks, 59 releases of Proton/Wine/DXVK/VKD3D |
| [Game DLLs Data](./game-dlls-data/) | `game-dlls-data-v0.0.1` | `.db.gz` | DLLs and winetricks for 13 games |
| [Releases Data](./releases-data/) | `releases-data-v0.0.1` | `.tar.gz` | Consolidated fallback metadata |

---

## Release System

### Tag format

```
<component-name>-v<major>.<minor>.<patch>
```

Examples: `catalogo-db-v0.0.1`, `installer-api-v0.0.7`

### Policy: never delete releases

Every release is immutable. If a version needs a fix, create a new one with an incremented version. Old releases remain available for rollback.

### How to publish a new version

1. Prepare the archive:
   - Code: `tar -czf <component>.tar.gz <files>`
   - Database: `gzip -kf <file>.db`

2. Create the GitHub release:
   - Tag: `<component>-v<new-version>`
   - Asset: the archive
   - Body: describe changes

3. Upload to GitLab (corresponding project ID)

4. Update `resonance.json` in this repository:
   - Increment component `version`
   - Update `tag`, `sha256` (compute with `sha256sum`), `size`
   - Commit and push

> **Warning**: if `resonance.json` is not updated, the app will never know a new version exists. Bootstrap only compares what is in `resonance.json`.

---

## For developers

### I want to contribute

Each component has its own folder with a README explaining schema, API, pipeline and publishing instructions:

- [`bootstrap/`](./bootstrap/)
- [`installer-api/`](./installer-api/)
- [`catalogo-db/`](./catalogo-db/)
- [`proton-data-db/`](./proton-data-db/)
- [`fork-catalog-data/`](./fork-catalog-data/)
- [`game-dlls-data/`](./game-dlls-data/)
- [`releases-data/`](./releases-data/)

### I want to report a bug

Open an issue with:
- Affected component
- Current version (release tag)
- Expected behavior
- Observed behavior

### I want to request a feature

Open an issue with `[FEATURE]` prefix in the title.

---

## 📓 Update history

> **Golden rule:** no version is ever deleted or overwritten. Each new release just adds a new entry to the history. Old versions remain accessible for rollback.

### How to update a component

1. **Prepare the archive** with your changes
2. **Calculate the SHA-256:**
   ```bash
   sha256sum <file>.tar.gz   # for code
   sha256sum <file>.db.gz    # for databases
   ```
   Copy the generated hash (e.g. `d74705009628e83737953691c5a7814212fa939673b2564be1473db8588a1fe2`)
3. **Create the release** on GitHub:
   - Tag: `<component>-v<new-version>` (e.g. `bootstrap-v0.0.2`)
   - Asset: the archive
   - Body: describe what changed
4. **Update `resonance.json`** in the root of this repository:
   - Increment the component's `version`
   - Update `tag`, `sha256` (hash from step 2), `size` (bytes)
5. **Commit and push**

### Version log

| Version | Date | What changed |
|---------|------|-------------|
| `bootstrap-v0.0.1` | 2026-06-15 | Initial unified ecosystem release. Includes: Bootstrap (orchestrator), Installer API v0.0.7, Catalogo DB, Proton Data DB, Fork Catalog, Game DLLs, Releases Data. |
| `installer-api-v0.0.7` | 2026-06-15 | Installer classification and extraction API with 7 extractors (archive, exe-companions, inno-std, iso, portable, sfx-nsis, wine-fallback). |
| `catalogo-db-v0.0.1` | 2026-06-15 | SQLite database with ~179k Steam + custom games, FTS5 schema, full metadata. |
| `proton-data-db-v0.0.1` | 2026-06-15 | ~81k Proton compatibility reports from ProtonDB. |
| `fork-catalog-data-v0.0.1` | 2026-06-15 | 30 cataloged forks (Proton, Wine, DXVK, VKD3D), 59 cached releases. |
| `game-dlls-data-v0.0.1` | 2026-06-15 | 13 mapped games, 10 cataloged DLLs with winetricks and overrides. |
| `releases-data-v0.0.1` | 2026-06-15 | Consolidated fallback metadata for the entire ecosystem. |

### Upcoming updates

When a new version is published:
- The old version **remains** as a GitHub release
- `resonance.json` is updated with the new entry
- Bootstrap compares versions on next startup and downloads the new one if needed
- Users who want to stay on the old version can disable auto-update (`bootstrap.json` → `auto_update: 0`)

---

<a name="espanol"></a>

# :es: The Bootstrap — Makai Forge

## Qué es

The Bootstrap es el sistema de inicio, actualización y aprovisionamiento de **Makai Forge**. Es lo primero que se ejecuta cuando la aplicación se abre. Garantiza que todos los componentes (catálogo de juegos, API de instaladores, datos de Proton, forks, DLLs) estén en la versión correcta antes de que la aplicación principal se inicie.

---

## Cómo Bootstrap gestiona las versiones

Bootstrap utiliza **dos archivos de metadatos** para decidir qué descargar:

### 1. `resonance.json` (remoto — en GitHub)

Se encuentra en la raíz de este repositorio. Es el manifiesto oficial con las versiones más recientes de cada componente.

```json
{
  "version": 1,
  "components": {
    "catalogo-db": {
      "version": 1,
      "tag": "catalogo-db-v0.0.1",
      "sha256": "af72a2fd...",
      "size": 55951414,
      "archive": "catalogo.db.gz"
    }
  }
}
```

### 2. `metadata.json` (local — en el disco del usuario)

Se almacena en la carpeta de datos de Makai Forge en la computadora del usuario. Registra qué versiones están instaladas localmente.

```json
{
  "catalogo-db": {
    "version": 0,
    "path": "/home/user/.config/makaiforger/resources/catalogo.db"
  }
}
```

---

## Flujo completo de inicio

### Primera ejecución (nunca descargó nada)

```
1. El usuario abre Makai Forge
   ↓
2. Bootstrap abre la ventana de splash
   ↓
3. Bootstrap descarga resonance.json de:
   ├── Primario: https://github.com/MakaiForge/the-bootstrap/raw/main/resonance.json
   └── Fallback: GitLab (project ID correspondiente)
   ↓
4. Bootstrap busca metadata.json local
   └── No existe (primera vez) → asume versión local = 0 para todo
   ↓
5. Para cada componente en resonance.json:
   ├── versión local (0) < versión remota (1)?
   │   └── SÍ → necesita descargar
   ├── 5.1 Descarga el archive del release del componente
   │   ├── Ej: https://github.com/MakaiForge/the-bootstrap/releases/download/catalogo-db-v0.0.1/catalogo.db.gz
   │   └── Guarda en: ~/.cache/makaiforger/downloads/catalogo.db.gz
   ├── 5.2 Calcula SHA-256 del archivo descargado
   ├── 5.3 Compara con el SHA-256 del resonance.json
   │   ├── ¿Coincide? → continúa
   │   └── ¿No coincide? → elimina y descarga de nuevo
   ├── 5.4 Extrae el archive
   │   ├── .tar.gz → extrae en ~/.config/makaiforger/resources/<componente>/
   │   └── .db.gz   → descomprime en ~/.config/makaiforger/resources/<archivo>.db
   ├── 5.5 Crea/actualiza metadata.json local:
   │   └── "catalogo-db": { "version": 1, "path": "..." }
   └── 5.6 Pasa al siguiente componente
   ↓
6. Todos los componentes actualizados
   ↓
7. Bootstrap cierra la ventana de splash
   ↓
8. La aplicación principal se abre
```

### Ejecuciones siguientes (ya tiene datos)

```
1. El usuario abre Makai Forge
   ↓
2. Bootstrap abre splash
   ↓
3. Bootstrap descarga resonance.json (remoto)
   ↓
4. Bootstrap lee metadata.json (local)
   ↓
5. Para cada componente:
   ├── catalogo-db: local version=1, remota version=1
   │   └── IGUAL → salta (ya actualizado)
   ├── installer-api: local version=8, remota version=8
   │   └── IGUAL → salta
   ├── proton-data-db: local version=1, remota version=2
   │   └── LOCAL < REMOTA → ¡necesita actualizar!
   │       ├── Descarga proton_data.db.gz del release proton-data-db-v0.0.2
   │       ├── Verifica SHA-256
   │       ├── Extrae
   │       └── metadata.json: proton-data-db.version = 2
   └── ...
   ↓
6. Bootstrap cierra splash
   ↓
7. La aplicación principal se abre
```

### Flujo visual simplificado

```
┌──────────────┐     ┌──────────────────┐     ┌───────────────────┐
│  Usuario     │     │  GitHub          │     │  Disco Local      │
│  abre app    │────>│  resonance.json  │────>│  metadata.json    │
└──────────────┘     └──────────────────┘     └───────────────────┘
                                                     │
                                                     ▼
                                            ┌──────────────────┐
                                            │  Compara         │
                                            │  versiones       │
                                            │  local vs remota │
                                            └──────────────────┘
                                                     │
                                      ┌──────────────┴──────────────┐
                                      ▼                             ▼
                               ┌─────────────┐             ┌──────────────┐
                               │  Versiones  │             │  Versiones   │
                               │  iguales    │             │  diferentes  │
                               │  → salta    │             │  → descarga  │
                               └─────────────┘             └──────┬───────┘
                                                                  │
                                                                  ▼
                                                         ┌────────────────┐
                                                         │  Descarga del  │
                                                         │  archive del   │
                                                         │  release       │
                                                         └────────┬───────┘
                                                                  │
                                                                  ▼
                                                         ┌────────────────┐
                                                         │  Verifica      │
                                                         │  SHA-256       │
                                                         └────────┬───────┘
                                                                  │
                                                                  ▼
                                                         ┌────────────────┐
                                                         │  Extrae y     │
                                                         │  actualiza    │
                                                         │  metadata     │
                                                         └────────────────┘
```

---

## Qué sucede después de una actualización

Cuando Bootstrap termina de descargar y extraer todo:

1. El archivo `metadata.json` local se actualiza con las nuevas versiones
2. El `resonance.json` descargado también se guarda en caché como referencia
3. En el **próximo inicio**:
   - Bootstrap descarga `resonance.json` nuevamente
   - Lee el `metadata.json` local
   - Compara: si las versiones coinciden, **no descarga nada**
   - Solo descarga cuando el `resonance.json` remoto tiene una versión mayor

### Ejemplo práctico

| Fecha | Acción | catalogo-db (local) | catalogo-db (remoto) | ¿Descargó? |
|-------|--------|--------------------|---------------------|------------|
| 01/01 | Primera ejecución | 0 | 1 | Sí |
| 02/01 | Abre de nuevo | 1 | 1 | No |
| 05/01 | Dev publica v0.0.2 | 1 | 2 | Sí |
| 06/01 | Abre de nuevo | 2 | 2 | No |
| 10/01 | Dev publica v0.0.3 | 2 | 3 | Sí |

### Auto-update: desactivar actualizaciones automáticas

Por defecto, Bootstrap descarga y actualiza todo automáticamente. Para desactivar este comportamiento:

1. Cree el archivo `bootstrap.json` en la carpeta de datos de Makai Forge:
   ```
   ~/.config/makaiforger/bootstrap.json
   ```
2. Con el contenido:
   ```json
   { "auto_update": 0 }
   ```
3. Con `1` (predeterminado) = actualiza automáticamente. Con `0` = nunca actualiza, usa los archivos locales como están.

Esto es útil para:
- Usuarios que quieren congelar una versión específica que funciona bien
- Desarrolladores modificando archivos manualmente sin que Bootstrap los sobrescriba
- Pruebas sin interferencia del actualizador

---

## Componentes

| Componente | Release | Formato | Descripción |
|---|---|---|---|
| [Bootstrap](./bootstrap/) | `bootstrap-v0.0.1` | `.tar.gz` | Orquestador de inicio, actualización y aprovisionamiento |
| [Installer API](./installer-api/) | `installer-api-v0.0.7` | `.tar.gz` | API de clasificación y extracción de instaladores |
| [Catálogo DB](./catalogo-db/) | `catalogo-db-v0.0.1` | `.db.gz` | ~179k juegos Steam + personalizados |
| [Proton Data DB](./proton-data-db/) | `proton-data-db-v0.0.1` | `.db.gz` | ~81k informes de compatibilidad Proton |
| [Fork Catalog Data](./fork-catalog-data/) | `fork-catalog-data-v0.0.1` | `.db.gz` | 30 forks, 59 releases de Proton/Wine/DXVK/VKD3D |
| [Game DLLs Data](./game-dlls-data/) | `game-dlls-data-v0.0.1` | `.db.gz` | DLLs y winetricks para 13 juegos |
| [Releases Data](./releases-data/) | `releases-data-v0.0.1` | `.tar.gz` | Metadatos consolidados de fallback |

---

## Sistema de Releases

### Formato de la etiqueta

```
<nombre-del-componente>-v<major>.<minor>.<patch>
```

Ejemplos: `catalogo-db-v0.0.1`, `installer-api-v0.0.7`

### Política: nunca eliminar releases

Cada release es inmutable. Si una versión necesita una corrección, cree una nueva con versión incremental. Los releases antiguos permanecen disponibles para rollback.

### Cómo publicar una nueva versión

1. Prepare el archive:
   - Código: `tar -czf <componente>.tar.gz <archivos>`
   - Base de datos: `gzip -kf <archivo>.db`

2. Cree el release en GitHub:
   - Tag: `<componente>-v<nueva-versión>`
   - Asset: el archive
   - Body: describa los cambios

3. Suba a GitLab (project ID correspondiente)

4. Actualice `resonance.json` en este repositorio:
   - Incremente `version` del componente
   - Actualice `tag`, `sha256` (calcule con `sha256sum`), `size`
   - Commit y push

> **Atención**: si no se actualiza `resonance.json`, la aplicación nunca sabrá que existe una nueva versión. Bootstrap solo compara lo que está en `resonance.json`.

---

## Para desarrolladores

### Quiero contribuir

Cada componente tiene su carpeta con un README que explica schema, API, pipeline e instrucciones de publicación:

- [`bootstrap/`](./bootstrap/)
- [`installer-api/`](./installer-api/)
- [`catalogo-db/`](./catalogo-db/)
- [`proton-data-db/`](./proton-data-db/)
- [`fork-catalog-data/`](./fork-catalog-data/)
- [`game-dlls-data/`](./game-dlls-data/)
- [`releases-data/`](./releases-data/)

### Quiero reportar un problema

Abra un issue con:
- Componente afectado
- Versión actual (tag del release)
- Comportamiento esperado
- Comportamiento observado

### Quiero solicitar una funcionalidad

Abra un issue con el prefijo `[FEATURE]` en el título.

---

## 📓 Historial de actualizaciones

> **Regla fundamental:** ninguna versión es jamás eliminada o sobrescrita. Cada nuevo release solo agrega una entrada nueva al historial. Las versiones antiguas continúan accesibles para rollback.

### Cómo actualizar un componente

1. **Prepare el archive** con los cambios deseados
2. **Calcule el SHA-256:**
   ```bash
   sha256sum <archivo>.tar.gz   # para código
   sha256sum <archivo>.db.gz    # para base de datos
   ```
   Copie el hash generado (ej: `d74705009628e83737953691c5a7814212fa939673b2564be1473db8588a1fe2`)
3. **Cree el release** en GitHub:
   - Tag: `<componente>-v<nueva-versión>` (ej: `bootstrap-v0.0.2`)
   - Asset: el archive
   - Body: describa lo que cambió
4. **Actualice `resonance.json`** en la raíz de este repositorio:
   - Incremente el `version` del componente
   - Actualice `tag`, `sha256` (el hash del paso 2), `size` (bytes)
5. **Commit y push**

### Registro de versiones

| Versión | Fecha | Qué cambió |
|---------|-------|-------------|
| `bootstrap-v0.0.1` | 2026-06-15 | Release inicial del ecosistema unificado. Incluye: Bootstrap (orquestador), Installer API v0.0.7, Catálogo DB, Proton Data DB, Fork Catalog, Game DLLs, Releases Data. |
| `installer-api-v0.0.7` | 2026-06-15 | API de clasificación y extracción con 7 extractors (archive, exe-companions, inno-std, iso, portable, sfx-nsis, wine-fallback). |
| `catalogo-db-v0.0.1` | 2026-06-15 | Base de datos SQLite con ~179k juegos Steam + personalizados, schema con FTS5, metadatos completos. |
| `proton-data-db-v0.0.1` | 2026-06-15 | ~81k informes de compatibilidad Proton de ProtonDB. |
| `fork-catalog-data-v0.0.1` | 2026-06-15 | 30 forks catalogados (Proton, Wine, DXVK, VKD3D), 59 releases en caché. |
| `game-dlls-data-v0.0.1` | 2026-06-15 | 13 juegos mapeados, 10 DLLs catalogadas con winetricks y overrides. |
| `releases-data-v0.0.1` | 2026-06-15 | Metadatos consolidados de fallback para todo el ecosistema. |

### Próximas actualizaciones

Cuando se publique una nueva versión:
- La versión antigua **sigue existiendo** como release en GitHub
- `resonance.json` se actualiza con la nueva entrada
- Bootstrap compara versiones en el próximo inicio y descarga la nueva si es necesario
- Los usuarios que quieran permanecer en la versión antigua pueden desactivar el auto-update (`bootstrap.json` → `auto_update: 0`)
