# Game DLLs Data

Banco de dados SQLite com recomendações de DLLs e comandos winetricks para jogos rodando via Proton no Makai Forger.

Parte do ecossistema **The Bootstrap** — atualizado automaticamente pelo `resource-manager.ts`.

## Schema

### `dll_catalog` — catálogo de DLLs conhecidas

| Coluna | Descrição |
|--------|-----------|
| `id` | Identificador (ex: `vcrun2022`, `mfplat`) |
| `dll` | Nome(s) do arquivo .dll |
| `impacto` | CRÍTICO / ALTO / MÉDIO / BAIXO |
| `winetricks` | Comando winetricks |
| `override` | WINEDLLOVERRIDES necessário |
| `descricao` | Explicação |

### `game_dlls` — mapeamento por jogo

| Coluna | Descrição |
|--------|-----------|
| `game_id` | Steam AppID ou `custom_<nome>` |
| `title` | Nome do jogo |
| `dlls` | JSON array de IDs (referencia `dll_catalog.id`) |
| `winetricks` | JSON array de comandos |
| `overrides` | Overrides específicos |

Dados atuais: 13 jogos mapeados, 10 DLLs catalogadas.

## Fluxo no app

```
Usuário → "Verificar DLLs" → checkGameDlls.ts → 
  ProtonRecommendationService.installGameDlls() →
  Python RPC → winetricks no prefixo
```

## Publicar

1. Atualize o SQLite
2. `gzip -kf game_dlls.db`
3. Release no GitHub com tag `game-dlls-data-v0.0.X`, asset: `game_dlls.db.gz`
4. GitLab fallback (project ID 83123076)
5. Atualize `resonance.json` no repositório `the-bootstrap`
