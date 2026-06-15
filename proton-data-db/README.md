# Proton Data DB

Banco de dados de compatibilidade Proton. ~81k jogos analisados do ProtonDB, com relatórios, notas, e recomendações de Proton por jogo.

Parte do ecossistema **The Bootstrap** — atualizado automaticamente pelo `resource-manager.ts` no Bootstrap.

## Publicar

1. Gere o `proton_data.db.gz`
2. Crie release no GitHub com tag `proton-data-db-v0.0.X`, asset: `proton_data.db.gz`
3. Envie pro GitLab (project ID 83081381)
4. Atualize `resonance.json` no repositório `the-bootstrap`
