# VirtueJS Bot — Documentação Técnica (DETALHADA)

Este documento descreve o funcionamento interno do bot, fluxos principais, pontos de extensão e onde alterar comportamentos.

---
## Visão Geral da Arquitetura
- **Entrada:** `src/index.ts` — cria `Client` do Discord, registra comandos, carrega eventos e inicia utilitários (ex.: `initializeRankMessage`).
- **Comandos:** cada comando em `src/commands/` exporta `data` (Slash builder) e `execute`.
- **Eventos:** `src/events/*` (ex.: `interactionCreate.ts`) roteia comandos e botões.
- **Jobs & serviços externos:** `src/jobs/` e `src/services/rucoy/` (scraping com `jsdom`).
- **Persistência:** JSONs simples em `src/data/` (players, xp-first, xp-history, mob-history).

---
## Fluxos Principais (passo-a-passo)

### 1) Inicialização (`src/index.ts`)
- Registra comandos via `deployCommands()` (usa `REST` e `Routes.applicationGuildCommands`).
- Carrega comandos `loadCommands()` e eventos `loadEvents()` para registrar handlers.
- No evento `ready`: chama `initializeRankMessage(client)` e — **se** `XP_SCAN_CONFIG.runOnStartup` for `true` — dispara `runXpFirstScan()` (não bloqueante).

### 2) Submissões e verificação (`/submit` → `src/commands/rank.ts`)
- Usuário envia stats e link de prova.
- O bot checa `ALLOWED_ROLE_IDS` e grava/atualiza o player em `src/data/players.json` via `savePlayers()`.
- Envia mensagem ao canal de verificação com botões `approve_{userId}` e `reject_{userId}`.
- Evento `interactionCreate` processa botões: marca `player.verified` e chama `updateRankMessage(client)` para atualizar embed fixo.

### 3) XP scraping e snapshot (`src/jobs/xp-first.job.ts`)
- Gera uma lista de meses com `generateFirstMonthOfEachYear()` (Janeiro de cada ano no intervalo configurado).
- Para cada mês, chama `fetchXpTop(year, month)` (scrape usando `jsdom` e parsing da tabela)
- Agrega os resultados em `{ name: xp }`, sobrescrevendo por último ano por padrão.
- Antes de salvar, carrega `xp-first.json` anterior e compara para detectar quedas de XP (mobs).
- Se `newXp < prevXp` → registra evento com `appendMobEvent(name, prevXp, newXp)` em `src/storage/mob-history.store.ts`.
- Escreve `xp-first.json` e um log detalhado em `xp-first.log` (linhas timestamped).

> Nota: `appendMobEvent` cria `src/data/mob-history.json` caso não exista. O formato de evento é: `{ date, lost, before, after }`.

### 4) Comando `/mobwins` (`src/commands/mobwins.ts`)
- Lê eventos via `getRecentMobEvents(limit)` e devolve um Embed com os eventos ordenados por data (mais recentes primeiro).
- Resposta é `ephemeral` (visível apenas para quem executou). Embeds grandes são truncados para evitar limites do Discord.

---
## Arquivos e responsabilidades (arquivo → função)
- `src/index.ts` — bootstrap do bot.
- `src/config/xp-scan.config.ts` — controla período, delays, e flags (ex.: `runOnStartup`, `verboseFirstScan`).
- `src/util/monthRange.ts` — `generateMonthRange()` (todos os meses) e `generateFirstMonthOfEachYear()` (apenas janeiro por ano).
- `src/services/rucoy/xpTop.service.ts` — scraping do highscores; retorna `[{name, xp}]`.
- `src/jobs/xp-first.job.ts` — rotina agregadora / detector de mobs / grava logs.
- `src/storage/mob-history.store.ts` — API para ler/gravar histórico de mobs.
- `src/commands/*.ts` — comandos do bot.
- `src/events/interactionCreate.ts` — roteador de comandos/botões e ações (savePlayers, updateRankMessage).
- `src/util/rankUpdater.ts` — cria/atualiza a mensagem do ranking fixo.

---
## Como modificar políticas importantes
- **Política de sobrescrita (quando mesmo nome aparece em anos diferentes):**
  - Atual: último ano processado prevalece (sobrescreve). Alterar no loop do job para escolher `Math.max(prev, entry.xp)` para manter o maior XP.
- **Registrar mobs apenas acima de X XP:** adicionar checagem `if (lost >= MIN_LOSS) ...` antes de `appendMobEvent`.
- **Alertas no canal:** dentro de `runXpFirstScan`, depois de detectar um mob, você pode usar um `client.channels.cache.get(ID)` e enviar um `channel.send()` com um embed (requer passar `client` para o job ou emitir um evento).

---
## Operação e deploy
- Ambiente: use um processo supervisor (systemd, PM2, etc.) ou execute via `bun` em um servidor.
- Registrar comandos: `deployCommands()` já roda na inicialização — assegure-se que `GUILD_ID` esteja configurado para desenvolvimento.
- Agendamento periódico: para rodar o job periodicamente, use cron ou um job scheduler (ex.: GitHub Actions, cron no servidor). Evite rodar com muito pouco delay para não sobrecarregar o site alvo.

---
## Debugging e logs
- `src/data/xp-first.log` — logs do processo `xp-first` (detalhado quando `verboseFirstScan=true`).
- `src/data/mob-history.json` — histórico de eventos detectados.
- `console` do processo — erros e mensagens principais.

---
## Testes e sugestões de melhorias
- Testes unitários podem ser adicionados para `xpTop.service` (mock HTML) e para `monthRange` util.
- Melhorar robustez do scraping: retry/backoff e limitação de requests.
- Tolerância a renomeações de jogadores: adicionar heurística de fuzzy-match para detectar mesmo jogador com nome diferente.

---
## Pistes rápidas (onde tocar para alterar coisas específicas)
- Mudar intervalo de anos: `src/config/xp-scan.config.ts` → `start`/`end`.
- Parar execuções automáticas na inicialização: `runOnStartup: false` no config.
- Mudar política de overwrite para manter maior XP: editar loop de agregação em `src/jobs/xp-first.job.ts`.

---
Se quiser, eu posso:
- adicionar alertas diretos no canal do Discord quando um mob for detectado; ou
- adicionar um resumo JSON/CSV com contagens por player e total por período.

Diga qual desses você prefere que eu implemente a seguir — posso abrir PR com as mudanças automaticamente. 🚀