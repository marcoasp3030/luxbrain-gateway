# PLANO_CURSOR.md — By Koji Gateway (VPS + Skyone Studio)

> **Instrução para o agente do Cursor:** este arquivo é o roteiro oficial do projeto. Leia inteiro antes de começar. Trabalhe **uma fase por vez**, na ordem. Ao fim de cada tarefa, rode a validação indicada. **Não avance de fase** se alguma validação falhar ou se houver um checkpoint "🛑 PARE" pendente. Registre tudo no `PROGRESS.md`.

---

## 1. Contexto

Agente de atendimento omnichannel (delivery e catering) para o By Koji, com unidades Morumbi, Einstein, Vila Nova Conceição e Pátio Guedala.

Divisão de responsabilidades:

| Camada | Responsável por |
|---|---|
| **VPS (este repositório)** | Receber webhooks dos canais, deduplicar, debounce, trava por conversa, histórico, carrinho, pedidos, cardápio, transbordo humano, envio ao canal, métricas |
| **Skyone Studio** | Agente de IA: prompt, base de conhecimento, Skills. Fluxo: Webhook → JavaScript → IF → AI Agent Call → Log → Retorno |

Canais desta fase: WhatsApp (uazapi para testes, Meta oficial depois), site, e-mail. iFood e 99Food são condicionais. **Sem integração com EPOC e Everest.**

```
Canal ──► /webhooks/* ──► Postgres (mensagens) + Redis (buffer)
                               │ debounce
                               ▼
                    Skyone (webhook do fluxo) ──► Retorno
                               ▼
                 envio ao canal + Postgres + tabela turnos
```

---

## 2. Regras obrigatórias (valem para todas as fases)

1. **Nunca rode testes contra o banco de produção.** Testes automatizados usam um banco cujo nome termina em `_test`. O `conftest.py` deve abortar se `DATABASE_URL` não terminar em `_test`.
2. **Não altere outros serviços da VPS.** Em especial a API DQE (`/dqe`) e as `location` existentes do Nginx. No Nginx, apenas **adicione** o bloco `/gateway/`. Antes de recarregar, rode `nginx -t`.
3. **Nunca exponha portas publicamente.** Gateway em `127.0.0.1:8100` e Postgres em `127.0.0.1:5433`. Acesso externo só pelo Nginx.
4. **Nunca coloque segredos no código, em logs ou no git.** Tudo em `.env`, que deve estar no `.gitignore`.
5. **Uvicorn com 1 worker.** O agendador do debounce roda dentro do processo. Não aumente workers sem antes mover o agendador para um processo separado.
6. **Preço nunca vem da LLM.** Todo preço e total vêm do Postgres.
7. **Mudança de schema:** a partir da Fase 4, use migrações versionadas em `app/migrations/NNN_descricao.sql`, aplicadas em ordem e registradas numa tabela `schema_migrations`. Não edite migração já aplicada.
8. **Comandos destrutivos** (`DROP`, `DELETE` sem `WHERE`, `docker compose down -v`, apagar volumes) exigem confirmação explícita do Marco.
9. **Não invente formatos de API externa.** Quando um payload de terceiro for incerto (uazapi, Meta, iFood), capture um exemplo real em `webhook_raw` e ajuste o código a partir dele.
10. Ao terminar cada fase, atualize o `PROGRESS.md` e faça commit com a mensagem `fase N: <resumo>`.

### Variáveis úteis para validação

```bash
cd ~/luxbrain-gateway   # ajuste para o caminho real
set -a; source .env; set +a
GW=http://127.0.0.1:8100
```

---

## 3. Formato do PROGRESS.md

Crie o arquivo na Fase 0 e acrescente uma entrada por fase:

```markdown
## Fase N — <nome> — <data>
- Status: concluída | bloqueada | em andamento
- Feito:
- Validações (comando → resultado):
- Métricas (se houver): p50 / p95 / erros
- Pendências / decisões para o Marco:
```

---

## FASE 0 — Implantação em modo simulado

**Objetivo:** gateway rodando na VPS com agente `mock` e envio `dry`.

### Tarefas
- [ ] Conferir pré-requisitos: `docker --version`, `docker compose version`, portas livres (`ss -ltnp | grep -E '8100|5433'` deve vir vazio).
- [ ] Criar `.gitignore` com `.env`, `__pycache__/` e `*.pyc`. Inicializar git, se ainda não houver.
- [ ] `cp .env.example .env` e gerar valores fortes para senhas e chaves (`openssl rand -hex 24`). Manter `SKYONE_MODE=mock` e `SEND_MODE=dry`.
- [ ] `docker compose up -d --build`
- [ ] Criar `PROGRESS.md`.

### Validação
```bash
curl -s $GW/health
# esperado: {"ok":true,"agente":"mock","envio":"dry"}

docker compose ps            # 3 serviços "running/healthy"
pip install httpx && python scripts/bench.py --url $GW --admin-key $ADMIN_KEY --clientes 5
# esperado: 5/5 respondidos; msgs_por_lote > 1; erros = 0
```

### Pronto quando
- Health OK, bench com 5/5 respondidos e 0 erros.
- Resultado do bench registrado no `PROGRESS.md` (é o **overhead da VPS**).

---

## FASE 1 — Exposição via Nginx e uazapi real

**Objetivo:** receber mensagens reais do WhatsApp de teste, ainda sem responder.

### Tarefas
- [ ] Adicionar no Nginx do domínio (sem alterar outros blocos):
  ```nginx
  location /gateway/ {
      proxy_pass http://127.0.0.1:8100/;
      proxy_set_header Host $host;
      proxy_read_timeout 120s;
  }
  ```
  Rodar `nginx -t` e só então `systemctl reload nginx`.
- [ ] 🛑 **PARE:** pedir ao Marco para configurar, na instância uazapi de teste, o webhook `https://<dominio>/gateway/webhooks/uazapi?s=<UAZAPI_WEBHOOK_SECRET>` (evento de mensagens) e enviar mensagens de texto, áudio, imagem e uma resposta manual pelo celular.
- [ ] Ler os payloads reais:
  ```bash
  docker compose exec postgres psql -U $POSTGRES_USER -d $POSTGRES_DB \
    -c "SELECT id, jsonb_pretty(payload) FROM webhook_raw ORDER BY id DESC LIMIT 5;"
  ```
- [ ] Ajustar `app/uazapi.py → normalizar()` ao formato real: telefone, texto, message_id, fromMe, enviada-pela-API, tipo de mídia.
- [ ] Confirmar na documentação da instância os endpoints de envio de texto e de presença ("digitando") e ajustar `enviar_texto()` e `digitando()`.
- [ ] Criar testes unitários de `normalizar()` usando os payloads reais **anonimizados** em `tests/fixtures/`.

### Validação
```bash
curl -s https://<dominio>/gateway/health
# na tabela mensagens: as mensagens de teste aparecem com origem 'cliente'
# a resposta manual pelo celular aparece com origem 'humano' e a conversa fica com bot_pausado_ate preenchido
# reenviar o mesmo payload → resposta {"duplicada": true}
```

### Pronto quando
- Texto, mídia e mensagem do atendente são classificados corretamente.
- Mensagens enviadas pelo próprio bot **não** pausam a conversa.

---

## FASE 2 — Agente Skyone ligado e linha de base de latência

**Objetivo:** medir o tempo real do agente, ainda sem enviar ao WhatsApp.

### Tarefas
- [ ] 🛑 **PARE:** o Marco cria no Skyone o fluxo `Webhook → JavaScript (valida token) → IF → AI Agent Call → Log → Retorno` e informa a URL do webhook. O Retorno deve devolver:
  ```json
  { "resposta": ["..."], "acao": "nenhuma", "motivo": null }
  ```
- [ ] No `.env`: `SKYONE_MODE=live`, `SKYONE_WEBHOOK_URL=<url>`, `SKYONE_TOKEN=<mesmo valor validado no JavaScript>`. Manter `SEND_MODE=dry`. Rodar `docker compose up -d gateway`.
- [ ] Conversa manual de teste:
  ```bash
  curl -s -X POST $GW/test/mensagem -H "X-Admin-Key: $ADMIN_KEY" \
    -H "Content-Type: application/json" -d '{"usuario":"marco","texto":"oi, vocês entregam no morumbi?"}'
  sleep 15; curl -s $GW/test/conversa/test:marco -H "X-Admin-Key: $ADMIN_KEY"
  ```
- [ ] Se o formato do Retorno não for reconhecido, ajustar `app/agente.py → _extrair()` com base no corpo real (logar o corpo bruto apenas em nível DEBUG).
- [ ] Rodar o bench com 1, 5 e 10 clientes.

### Validação
```bash
python scripts/bench.py --url $GW --admin-key $ADMIN_KEY --clientes 5
curl -s "$GW/metricas?ultimos=50" -H "X-Admin-Key: $ADMIN_KEY"
```

### Pronto quando
- 0 erros com 5 clientes.
- `agente_p50`, `agente_p95` e `total_p95` registrados no `PROGRESS.md`.
- 🛑 **PARE se `total_p95` > 15 s:** reportar ao Marco antes de seguir (pode exigir prompt menor, modelo mais rápido ou resposta assíncrona).

---

## FASE 3 — Conversa humanizada no WhatsApp

**Objetivo:** atendimento real pelo celular, robusto a falhas.

### Tarefas
- [ ] **Infra de testes:** `docker-compose.test.yml` ou banco `bykoji_test` separado; `tests/conftest.py` que aborta se o banco não terminar em `_test`; `pytest` + `pytest-asyncio`.
- [ ] Testes do pipeline com agente mock:
  - [ ] 3 mensagens em menos de `DEBOUNCE_S` geram 1 chamada ao agente.
  - [ ] Teto `MAX_WAIT_S` respeitado mesmo com mensagens contínuas.
  - [ ] Mensagem que chega durante o processamento entra no lote seguinte, nunca em paralelo.
  - [ ] Conversa pausada não chama o agente.
  - [ ] Falha ou timeout do agente envia `FALLBACK` e grava `status='erro'`.
  - [ ] Agente lento (> `SLOW_NOTICE_S`) envia `AVISO_DEMORA` uma única vez.
- [ ] **Retomada do bot:** quando a pausa expirar, o bot volta sozinho. Adicionar também um comando do atendente (ex.: mensagem `#bot` enviada pelo celular) que retoma imediatamente e não é encaminhado ao agente.
- [ ] **Alerta de transbordo:** quando `acao = transferir_humano`, enviar aviso para um número interno (`ALERTA_TELEFONE` no `.env`) com nome, telefone e motivo.
- [ ] **Recuperação após reinício:** na subida do app, reagendar conversas com `buf:*` pendente no Redis.
- [ ] `SEND_MODE=live` somente para a instância de teste.
- [ ] 🛑 **PARE:** o Marco testa pelo celular com o roteiro de 30 a 50 conversas e ajusta prompt e base de conhecimento no Skyone.

### Validação
```bash
docker compose exec gateway pytest -q          # tudo verde
curl -s "$GW/metricas?ultimos=100" -H "X-Admin-Key: $ADMIN_KEY"
```

### Pronto quando
- Testes verdes.
- Mensagens picadas recebem 1 resposta coerente no celular.
- Atendente assume e o bot silencia; `#bot` retoma.
- Reiniciar o container com mensagens no buffer não perde a resposta.

---

## FASE 4 — Cardápio, carrinho e pré-pedido

**Objetivo:** cliente monta e fecha pedido pelo WhatsApp; a unidade confirma.

> Sem EPOC, o pedido é um **pré-pedido**: só vale após a confirmação da unidade.

### Tarefas
- [ ] Criar o mecanismo de migrações (Regra 7) e as migrações:
  - `carrinhos` (conversation_id, unidade, modalidade `delivery|retirada`, endereço, horário, observação, status)
  - `carrinho_itens` (carrinho_id, item_id, quantidade, preço unitário **copiado do cardápio**, observação)
  - `pedidos` (número curto legível, conversation_id, unidade, itens jsonb, total, status `aguardando_loja | confirmado | recusado | cancelado`, timestamps)
- [ ] 🛑 **PARE:** obter do Marco o cardápio real (CSV), números de WhatsApp de cada unidade, formas de pagamento e regras de entrega.
- [ ] Endpoints para Skills (header `X-API-Key`), todos exigindo `conversation_id` válido e ativo:

  | Método e rota | Função |
  |---|---|
  | `GET /api/cardapio/busca` | já existe |
  | `GET /api/carrinho/{conv}` | itens, subtotal, dados faltantes |
  | `POST /api/carrinho/{conv}/itens` | adiciona item por `item_id` e devolve o carrinho atualizado |
  | `PATCH /api/carrinho/{conv}/itens/{id}` | altera quantidade ou observação |
  | `DELETE /api/carrinho/{conv}/itens/{id}` | remove item |
  | `PUT /api/carrinho/{conv}/entrega` | unidade, modalidade, endereço, horário |
  | `POST /api/pedidos` | fecha o carrinho; recusa se faltar dado obrigatório |
  | `GET /api/pedidos/{numero}` | status do pedido |

- [ ] Toda resposta de escrita devolve o **carrinho completo atualizado**, para o agente não precisar de outra chamada.
- [ ] Ao fechar o pedido: enviar resumo para o WhatsApp da unidade. A unidade responde `OK <numero>` ou `NÃO <numero> <motivo>`, o status é atualizado e o cliente é avisado.
- [ ] Enviar o `carrinho` real no payload ao Skyone (hoje vai vazio).
- [ ] Documentar em `docs/SKILLS.md` a especificação de cada Skill para o Marco criar no Skyone: nome, descrição para o agente, parâmetros e exemplo de retorno.
- [ ] Testes: preço sempre do banco; item inativo recusado; `conversation_id` inexistente → 404; pedido sem endereço em delivery → 422; total = soma dos itens.

### Validação
```bash
docker compose exec gateway pytest -q
# fluxo manual via curl: adicionar 2 itens → alterar → fechar → unidade confirma → cliente recebe aviso
```

### Pronto quando
- Pedido completo pelo celular chega à unidade com o total correto.
- Nenhum preço da resposta do agente diverge do banco no roteiro de testes.

---

## FASE 5 — Áudio e imagem

### Tarefas
- [ ] Baixar mídia pelo uazapi (confirmar o endpoint com um payload real).
- [ ] Áudio → transcrição (Whisper via API; chave em `.env`) → texto entra no mesmo debounce, com prefixo indicando que veio de áudio.
- [ ] Imagem → enviar ao Skyone como URL temporária ou base64 em campo `midia` do payload. Combinar com o Marco o formato que o fluxo aceita (Processamento OCR).
- [ ] Limites: tamanho máximo, timeout de transcrição e fallback ("não consegui ouvir seu áudio, pode escrever?").
- [ ] Registrar a duração da transcrição em `turnos` (nova coluna via migração).

### Pronto quando
- Pedido feito por áudio funciona igual ao feito por texto.
- Falha na transcrição gera resposta amigável, não silêncio.

---

## FASE 6 — Site e e-mail

### Tarefas
- [ ] `POST /webhooks/site` com sessão por visitante (`site:<visit_id>`), CORS restrito ao domínio do cliente e limite de requisições por IP.
- [ ] Resposta para o site: polling `GET /site/mensagens/{visit_id}?desde=<id>` (mais simples que WebSocket nesta fase).
- [ ] Widget JS mínimo de exemplo em `widget/`.
- [ ] E-mail: worker IMAP (processo separado no compose), a cada 2–5 min, conversa `email:<endereco>`, sem debounce, resposta via SMTP com o assunto `Re:` original; ignorar no-reply, auto-respostas e listas.
- [ ] Vincular contato entre canais quando o cliente informar telefone (tabela `contato_vinculos`).

### Pronto quando
- Mesmo agente responde nos três canais com o mesmo comportamento.

---

## FASE 7 — WhatsApp oficial (Meta)

### Tarefas
- [ ] 🛑 **PARE:** o Marco providencia app na Meta, número de produção, tokens e templates aprovados.
- [ ] `GET /webhooks/meta` (verificação `hub.challenge`) e `POST /webhooks/meta` com validação da assinatura `X-Hub-Signature-256`.
- [ ] Responder 200 imediatamente e processar depois; deduplicar por id da mensagem.
- [ ] Adaptador de envio Meta; controle da janela de 24h (fora dela, só template).
- [ ] Download de mídia com token.
- [ ] O canal (uazapi ou Meta) passa a ser escolhido por configuração, sem mudar o pipeline nem o Skyone.

### Pronto quando
- O mesmo roteiro de testes da Fase 3 passa pelo número oficial.

---

## FASE 8 — iFood e 99Food (condicional)

> Só iniciar após o Marco confirmar o credenciamento e o tipo de acesso (eventos por webhook ou polling; existência ou não de API de mensagens com o cliente).

### Tarefas
- [ ] Capturar eventos reais em `webhook_raw` antes de modelar.
- [ ] Registrar pedidos externos em `pedidos` com `origem = ifood | 99food`.
- [ ] Avisar a unidade e, se o cliente tiver autorizado, enviar status pelo WhatsApp.

---

## FASE 9 — Piloto e produção

### Tarefas
- [ ] Backup diário do Postgres (`pg_dump` + rotação de 7/30 dias) e **teste de restauração** documentado.
- [ ] Retenção de dados (LGPD): job que anonimiza ou apaga mensagens e `webhook_raw` após o prazo definido pelo cliente.
- [ ] Logs estruturados com `request_id`, sem conteúdo de mensagens em nível INFO.
- [ ] Alerta (WhatsApp interno ou e-mail) quando: `/health` falhar, taxa de erro > 5% em 15 min ou `total_p95` > 20 s.
- [ ] Painel simples `GET /admin/painel` (HTML) com métricas do dia e conversas pausadas.
- [ ] 🛑 **PARE:** piloto em uma unidade, horário limitado, com atendente acompanhando. Revisão diária das conversas com o Marco.

### Critérios de go-live
- Transbordo abaixo de ~15% das conversas.
- Zero pedidos com preço divergente durante o piloto.
- Restauração de backup testada.

---

## Apêndice A — Contrato VPS → Skyone

```json
{
  "token": "<SKYONE_TOKEN>",
  "request_id": "uuid",
  "conversation_id": "wa:5511999999999",
  "canal": "wa",
  "mensagem": "texto consolidado do lote",
  "historico": [{"role": "user|assistant", "content": "..."}],
  "cliente": {"nome": "Ana", "unidade_preferida": "Morumbi"},
  "carrinho": {"itens": [], "total": 0}
}
```

Retorno esperado:

```json
{ "resposta": ["balão 1", "balão 2"], "acao": "nenhuma | transferir_humano | pedido_fechado", "motivo": null }
```

## Apêndice B — Metas de latência

| Métrica | Meta |
|---|---|
| Overhead da VPS (modo mock, sem contar o mock) | < 300 ms |
| `total_p50` (live) | ≤ 10 s |
| `total_p95` (live) | ≤ 15 s |
| Taxa de erro | < 1% |

## Apêndice C — Checkpoints que dependem do Marco

| Fase | O que é necessário |
|---|---|
| 1 | Configurar webhook no uazapi e enviar mensagens de teste |
| 2 | Fluxo do agente no Skyone e URL do webhook |
| 3 | Rodada de testes no celular e ajustes de prompt |
| 4 | Cardápio real, contatos das unidades, pagamento, regras de entrega; criação das Skills no Skyone |
| 5 | Formato de mídia aceito pelo fluxo Skyone |
| 7 | Conta Meta, número e templates |
| 8 | Credenciamento iFood / 99Food |
| 9 | Prazo de retenção (LGPD) e escolha da unidade piloto |
