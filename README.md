# The Bootstrap — Makai Forger

## O que é

The Bootstrap é o sistema de inicialização, atualização e provisionamento do **Makai Forger**. Ele é a primeira coisa que roda quando o aplicativo é iniciado.

Sua função é simples: garantir que todos os componentes do ecossistema estejam na versão correta antes do app principal abrir. Ele baixa, verifica, extrai e gerencia cada recurso — catálogo de jogos, API de instaladores, dados de compatibilidade Proton, forks, DLLs — tudo de forma autônoma.

## Arquitetura do ecossistema

O Makai Forger é dividido em **7 componentes**, cada um versionado independentemente, mas todos entregues através deste repositório único:

| Componente | Release | Formato | Função |
|---|---|---|---|
| **Bootstrap** | `bootstrap-v0.0.1` | `.tar.gz` | Orquestrador — baixa, verifica e extrai todos os outros componentes |
| **Installer API** | `installer-api-v0.0.7` | `.tar.gz` | API de classificação e extração de instaladores de jogos |
| **Catálogo DB** | `catalogo-db-v0.0.1` | `.db.gz` | Banco SQLite com ~179k jogos Steam + custom |
| **Proton Data DB** | `proton-data-db-v0.0.1` | `.db.gz` | ~81k relatórios de compatibilidade Proton |
| **Fork Catalog Data** | `fork-catalog-data-v0.0.1` | `.db.gz` | Catálogo de forks Proton/Wine/DXVK/VKD3D |
| **Game DLLs Data** | `game-dlls-data-v0.0.1` | `.db.gz` | Recomendações de DLLs e winetricks por jogo |
| **Releases Data** | `releases-data-v0.0.1` | `.tar.gz` | Metadados consolidados de fallback |

## Fluxo completo de inicialização

Quando o usuário abre o Makai Forger, a sequência abaixo acontece:

```
1. App inicia
   ↓
2. Bootstrap abre (janela de splash/progresso)
   ↓
3. Bootstrap baixa resonance.json
   ├── URL: https://github.com/MakaiForger/the-bootstrap/raw/main/resonance.json
   ├── Fallback: GitLab (project ID correspondente)
   ↓
4. Para cada componente listado no resonance.json:
   ├── 4.1 Lê versão local (do disco)
   ├── 4.2 Compara com versão remota (do resonance.json)
   ├── 4.3 Se local < remota:
   │   ├── Baixa o archive do componente
   │   │   ├── Fonte primária: GitHub Release asset
   │   │   └── Fallback: GitLab package
   │   ├── Calcula SHA-256 do arquivo baixado
   │   ├── Compara com o SHA-256 do resonance.json
   │   │   ├── Confere? → extrai no diretório do componente
   │   │   └── Não confere? → rejeita, baixa novamente
   │   └── Atualiza metadata local
   └── 4.4 Se local == remota:
       └── Pula (já está atualizado)
   ↓
5. Bootstrap inicializa a aplicação principal
   ↓
6. App principal abre
```

### Detalhamento de cada etapa

#### 3. resonance.json — o manifesto

O `resonance.json` é o arquivo central que coordena todo o ecossistema. Ele contém:

```json
{
  "version": 1,
  "components": {
    "bootstrap": {
      "version": 1,
      "tag": "bootstrap-v0.0.1",
      "sha256": "a523c6f970a050c1fef45d32ebdf87949b57d3ac20b397e5bb05119cee5caf7f",
      "size": 2396160,
      "archive": "bootstrap.tar.gz"
    }
  }
}
```

- **version**: versão numérica do componente (1, 2, 3...)
- **tag**: nome da release no GitHub
- **sha256**: hash do archive compactado (usado pra verificar integridade)
- **size**: tamanho em bytes
- **archive**: nome do arquivo

#### 4.3 SHA-256 — verificação de integridade

Cada archive tem seu SHA-256 registrado no `resonance.json`. O Bootstrap calcula o hash do arquivo baixado e compara. Se não bater, o download é rejeitado e tentado novamente. Isso garante que nenhum archive corrompido ou adulterado seja extraído.

## Sistema de Releases

Cada componente tem seu próprio release dentro deste repositório. O formato da tag é:

```
<nome-do-componente>-v<major>.<minor>.<patch>
```

Exemplos:
- `bootstrap-v0.0.1`
- `installer-api-v0.0.7`
- `catalogo-db-v0.0.1`

Cada release contém **um único archive** como asset. A relação é 1:1 — um release, um componente, um arquivo.

### Política: nunca apagar releases

> Releases são imutáveis. Nunca delete ou sobrescreva um release existente.

Cada release representa um ponto na história do componente. Se uma versão precisa ser corrigida, crie uma nova com versão incremental. Releases antigos continuam disponíveis para rollback.

### Como publicar um novo release

1. Prepare o archive do componente:
   - Código: `tar -czf <componente>.tar.gz <arquivos>`
   - Banco de dados: `gzip -kf <arquivo>.db`

2. Crie o release no GitHub:
   ```
   Tag: <componente>-v<nova-versão>
   Asset: o archive gerado
   Body: descreva as mudanças
   ```

3. Faça upload do mesmo archive no GitLab (project ID correspondente)

4. Atualize o `resonance.json` na raiz deste repositório:
   - Incremente o `version` do componente
   - Atualize `tag` para a nova tag
   - Atualize `sha256` (calcule com `sha256sum <archive>`)
   - Atualize `size` (bytes)
   - Commit e push

5. **Importante**: se você não atualizar o `resonance.json`, o aplicativo nunca saberá que existe uma versão nova e nunca fará o download.

## Para desenvolvedores

### Quero contribuir com um componente

Cada componente tem sua pasta neste repositório com README próprio explicando:

- Schema (para bancos de dados)
- API (para código)
- Pipeline de geração
- Instruções de publicação

Entre na pasta do componente desejado e leia o README:

- [`bootstrap/`](./bootstrap/) — código TypeScript do orquestrador
- [`installer-api/`](./installer-api/) — API de classificação e extração
- [`catalogo-db/`](./catalogo-db/) — banco de dados do catálogo
- [`proton-data-db/`](./proton-data-db/) — dados de compatibilidade Proton
- [`fork-catalog-data/`](./fork-catalog-data/) — forks Proton/Wine/DXVK/VKD3D
- [`game-dlls-data/`](./game-dlls-data/) — DLLs e winetricks
- [`releases-data/`](./releases-data/) — metadados consolidados

### Quero reportar um problema

Abra uma issue neste repositório. Especifique:
- Qual componente
- Versão atual (tag do release)
- Comportamento esperado
- Comportamento observado

### Quero solicitar uma funcionalidade

Abra uma issue com o prefixo `[FEATURE]` no título. Descreva a funcionalidade e em qual componente ela se aplica.
