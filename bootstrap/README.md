# Bootstrap

Orquestrador de inicialização do Makai Forger. Gerencia o ciclo de vida do app: verificação de atualizações, download de componentes, extração e inicialização da aplicação principal.

**Tecnologia:** TypeScript

## Estrutura

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
