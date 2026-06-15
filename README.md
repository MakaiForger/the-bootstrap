# The Bootstrap — Makai Forger

Sistema unificado de atualização e provisionamento do **Makai Forger**. Quando o app inicia, o Bootstrap abre, baixa o `resonance.json`, compara versões, e atualiza cada componente que precisa.

## Componentes

| Componente | Versão | O que faz |
|---|---|---|
| [**Bootstrap**](./bootstrap/) | `v0.0.1` | Orquestrador de inicialização, atualização e provisionamento |
| [**Installer API**](./installer-api/) | `v0.0.7` | Classifica e extrai instaladores de jogos (7z, innoextract, Wine) |
| [**Catálogo DB**](./catalogo-db/) | `v0.0.1` | ~179k jogos Steam + custom com metadados, Proton, preços |
| [**Proton Data DB**](./proton-data-db/) | `v0.0.1` | Dados de compatibilidade Proton (~81k jogos do ProtonDB) |
| [**Fork Catalog Data**](./fork-catalog-data/) | `v0.0.1` | Catálogo de forks Proton/Wine/DXVK/VKD3D (30 forks, 59 releases) |
| [**Game DLLs Data**](./game-dlls-data/) | `v0.0.1` | DLLs recomendadas + winetricks por jogo (13 jogos, 10 DLLs) |
| [**Releases Data**](./releases-data/) | `v0.0.1` | Metadados consolidados das releases do ecossistema |

Cada componente tem sua própria pasta com documentação completa, schema e instruções de publicação. Basta entrar na pasta do componente que você quer ajudar e ler o README.

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

## Política de Releases

> **Nunca apague um release existente.**

Cada release representa um ponto imutável na história de um componente. Para corrigir ou evoluir, crie um **novo release** com versão incremental.

### ⚠️ Ao publicar

1. Publique o release no GitHub (tag + asset)
2. Faça upload do asset para o GitLab
3. Atualize o `resonance.json` com a nova versão, SHA256 e size
4. Commit e push

O aplicativo sempre baixa o `resonance.json` na inicialização. Se ele não for atualizado, o app nunca saberá que existe uma versão nova.
