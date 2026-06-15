# Catálogo DB

Banco de dados mestre de jogos do Makai Forger. SQLite com ~179k jogos do catálogo público Steam, mais jogos custom (Genshin Impact, NIKKE, etc.).

Parte do ecossistema **The Bootstrap** — atualizado automaticamente pelo `resource-manager.ts` no Bootstrap.

## Schema

```sql
CREATE TABLE games (
    objectId             TEXT PRIMARY KEY,    -- 'steam_<appid>' | 'custom_<nome>'
    title                TEXT NOT NULL,
    shop                 TEXT NOT NULL,       -- 'steam' | 'custom'
    genres               TEXT,                -- JSON array
    libraryImageUrl      TEXT,                -- Capa (460x215)
    libraryHeroImageUrl  TEXT,                -- Fundo (1920x720)
    iconUrl              TEXT,                -- Ícone (256x256)
    shortDescription     TEXT,
    developer            TEXT,
    publisher            TEXT,
    minimum              TEXT,                -- Requisitos mínimos
    recommended          TEXT,                -- Requisitos recomendados
    releaseYear          TEXT,
    release_date         TEXT,
    recommendedProton    TEXT,                -- Proton recomendado
    protonConfidence     TEXT,                -- high/medium/low
    protonSource         TEXT,                -- Fonte da recomendação
    protonFallback       TEXT,
    protonAlternatives   TEXT,                -- JSON: alternativas ProtonDB
    downloadSources      TEXT,                -- JSON: nomes das fontes
    downloads            TEXT,                -- JSON: links oficiais (jogos gratuitos)
    screenshots          TEXT,                -- JSON: URLs
    movies               TEXT,                -- JSON: vídeos
    pcRequirements       TEXT,                -- JSON: requisitos estruturados
    estimated_owners     INTEGER
);

CREATE VIRTUAL TABLE games_fts USING fts5(
    objectId UNINDEXED, title, genres, developer, publisher,
    content='games', content_rowid='rowid'
);
```

## Pipeline

```bash
node scripts/fetch-games.cjs              # Buscar metadados Steam
node scripts/populate-download-games.cjs   # Popular SQLite
node scripts/run-migration.cjs             # Mesclar jogos custom
gzip -kf resources/catalogo.db             # Compactar
```

## Publicar

1. Gere o `catalogo.db.gz`
2. Crie release no GitHub com tag `catalogo-db-v0.0.X`, asset: `catalogo.db.gz`
3. Envie pro GitLab (project ID 83081380)
4. Atualize `resonance.json` no repositório `the-bootstrap`
