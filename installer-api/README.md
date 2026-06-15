# Installer API

API modular de classificação e extração de instaladores de jogos. Detecta automaticamente o tipo do instalador e usa extração nativa (Linux) sempre que possível, com Wine apenas como fallback.

**Tecnologia:** JavaScript (Node.js)

## Arquitetura

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

## API

### `analyze(sourcePath, gameId?, gameTitle?)`

Classifica um instalador e retorna metadados.

```js
const { analyze } = require('./index');
const info = analyze('/path/to/setup.exe', 'game-123', 'Meu Jogo');
// { type, method, needsWine, confidence, gameName, companionArchives, ... }
```

### `extract(installInfo, options)`

Extrai um instalador com base no resultado do analyze.

```js
const { extract } = require('./index');
const result = extract(info, {
  destPath: '/home/user/Games/MeuJogo/pfx/drive_c',
  protonPath: '/path/to/proton',
  source: 'catalog',
  onProgress: (msg) => console.log(msg),
});
// { success, destDir, candidates, registryNeeded, error }
```

## Tipos de instalador

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

## CLI

```bash
node index.js --analyze setup.exe
node index.js --extract setup.exe --dest /destino --source catalog
```

## Publicar

1. `tar -czf installer-api.tar.gz index.js classifier.js utils.js overrides-loader.js overrides.json extractors/ overrides/`
2. Crie release no GitHub com tag `installer-api-v0.0.X`, asset: `installer-api.tar.gz`
3. Envie pro GitLab (project ID 83101354)
4. Atualize `resonance.json` no repositório `the-bootstrap`
