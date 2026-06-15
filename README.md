# The Bootstrap — Makai Forger

Sistema unificado de atualização e provisionamento do **Makai Forger**. Centraliza todos os componentes do ecossistema em um único repositório, com versionamento por releases individuais.

## Componentes

| Componente | Versão | Arquivo |
|---|---|---|
| **Bootstrap** | `v0.0.1` | `bootstrap.tar.gz` |
| **Installer API** | `v0.0.7` | `installer-api.tar.gz` |
| **Catálogo DB** | `v0.0.1` | `catalogo.db.gz` |
| **Proton Data DB** | `v0.0.1` | `proton_data.db.gz` |
| **Fork Catalog Data** | `v0.0.1` | `fork_catalog.db.gz` |
| **Game DLLs Data** | `v0.0.1` | `game_dlls.db.gz` |
| **Releases Data** | `v0.0.1` | `releases.tar.gz` |

## Política de Releases

> **Nunca apague um release existente.**

Cada release representa um ponto imutável na história de um componente. Para corrigir ou evoluir, crie um **novo release** com versão incremental. Releases antigos continuam disponíveis para rollback e rastreabilidade.

### Versionamento

```
<nome-do-componente>-v<major>.<minor>.<patch>
```

Exemplo: `installer-api-v0.0.7`, `catalogo-db-v0.0.1`

### Como criar um novo release

1. Compacte o componente: `tar -czf <componente>.tar.gz <src>/`
2. Crie uma release via GitHub:
   - **Tag:** `<nome>-v<versão>`
   - **Asset:** o arquivo compactado
3. Atualize o `resonance.json` com o novo SHA-256 e versão

## Estrutura do Repositório

```
/
├── README.md
├── resonance.json          ← Metadados de todos os componentes
├── background.png          ← Identidade visual do Bootstrap
├── *.tar.gz / *.db.gz      ← Arquivos compactados (assets)
├── bootstrap/              ← Código-fonte do Bootstrap (TypeScript)
│   ├── setup-window.ts
│   ├── resource-manager.ts
│   ├── venv.ts
│   ├── update-manager.ts
│   └── autoupdater/
└── installer-api/          ← Código-fonte da Installer API (JavaScript)
    ├── index.js
    ├── classifier.js
    ├── utils.js
    ├── overrides-loader.js
    ├── overrides.json
    ├── extractors/
    └── overrides/
```

## Licença

Todos os direitos reservados — Makai Forger.
