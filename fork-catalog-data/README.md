# Fork Catalog Data

Banco de dados SQLite com catálogo completo de forks de Proton, Wine, DXVK e VKD3D — versões, metadados e URLs de download. Serve como cache local para evitar chamadas excessivas às APIs.

Parte do ecossistema **The Bootstrap** — atualizado automaticamente pelo `resource-manager.ts`.

## Schema

### `fork_tools` — definição de cada fork (30 entries)

| Coluna | Descrição |
|--------|-----------|
| `id` | Identificador único (ex: `proton-ge`) |
| `title` | Nome legível |
| `category` | proton / wine / dxvk / vkd3d |
| `endpoint` | URL da API de releases |
| `directory_name_format` | Template da pasta (`$version`) |
| `keywords` | JSON array de sinônimos |
| `support_latest` | 1 se suporta latest |
| `asset_position` | Posição do asset |
| `asset_filter` | Regex para filtrar assets |
| `sort_order` | Ordem de exibição |

### `fork_releases` — cache de versões (59 releases)

| Coluna | Descrição |
|--------|-----------|
| `fork_id` | Referencia `fork_tools.id` |
| `versao` | Nome da versão |
| `tag` | Tag do release |
| `download_url` | URL direta do asset |
| `tamanho_bytes` | Tamanho do arquivo |
| `publicacao` | Data ISO |
| `body` | Notas de release |

### `fork_overview` — resumo estático por fork

## Publicar

1. Atualize o SQLite com novos forks/releases
2. `gzip -kf fork_catalog.db`
3. Release no GitHub com tag `fork-catalog-data-v0.0.X`, asset: `fork_catalog.db.gz`
4. GitLab fallback (project ID 83110945)
5. Atualize `resonance.json` no repositório `the-bootstrap`
