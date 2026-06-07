# VirtueJS Bot

Um bot do Discord para gerenciar rankings, histórico de XP (scraping de Rucoy Online), submissões manuais de stats e monitoramento de guild. Rápido, simples e focado em fornecer um top e histórico de perdas de XP ("mobs").

## Funcionalidades ✅
- Submissão de stats com prova (/submit) e aprovação manual via botões
- Mensagem fixa com ranking atualizável (`rankUpdater`) ✅
- Comandos para visualizar rankings: `/viewrank`, `/viewallrank` (paginação)
- Job de scraping de highscores do Rucoy (`xp-first`) para snapshots anuais (Janeiro de cada ano)
- Detecção automática de "mob wins" (perda de XP) comparando snapshots, com histórico
- Monitoramento de guild (guild spy) que notifica quando players vão online

## Instalação e execução ▶️
Requisitos: bun (usado nos scripts), Node 18+

1. Copie as variáveis de ambiente necessárias em um `.env`:
   - `DISCORD_TOKEN` (token do bot)
   - `CLIENT_ID` (app client id)
   - `GUILD_ID` (id do guild para registrar comandos, opcional em produção global)

2. Rodar em desenvolvimento:
   - `bun run src/index.ts` (ou `bun run dev` se preferir script `dev`)

3. Registrar e rodar o job manualmente:
   - `bun run xp-first` (executa a rotina que cria `src/data/xp-first.json` com `{ "nome": XP }`)

## Principais comandos
- `/submit level:... melee:... proof:...` — envia stats e prova para verificação
- `/viewrank categoria:level|melee|pally|mage|defense` — mostra Top 25
- `/viewallrank` — pagina entre categorias
- `/guildspy guild:<nome> channel:<canal>` — inicia monitoramento de uma guild
- `/mobwins [limit]` — mostra os últimos eventos de perda de XP (mob wins)

> Observação: comandos são registrados automaticamente em ambiente de desenvolvimento via `deployCommands()` na inicialização.

## Arquivos importantes 📂
- `src/data/xp-first.json` — snapshot atual com `{ nome: xp }` (gerado por `xp-first` job)
- `src/data/xp-first.log` — log detalhado do job
- `src/data/mob-history.json` — histórico de eventos de perda de XP
- `src/data/players.json` — submissões e verificações de players

## Configuração
Ajustes em `src/config/xp-scan.config.ts`:
- `start`/`end` — período do scraping
- `firstMonthEachYear` — quando true, o job pega apenas Janeiro de cada ano
- `runOnStartup` — iniciar o job automaticamente na inicialização do bot
- `verboseFirstScan` e `logFile` — controlar logs