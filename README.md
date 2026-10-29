# Servidor de WhatsApp — Kairós (v5.2)

Este é o "motor" que mantém o WhatsApp da empresa conectado 24 horas por dia.
Ele recebe as mensagens dos clientes, responde pela IA, envia as respostas dos
atendentes e dispara as cobranças e confirmações de pagamento.

> ⚠️ **Ele precisa ficar ligado o tempo todo e precisa de um disco próprio.**
> Hospedagens que "dormem" por inatividade ou que apagam o disco a cada deploy
> (Vercel, Netlify, Cloudflare, funções serverless) **não servem** — a sessão do
> WhatsApp morre e você precisa escanear o QR Code de novo toda hora.

---

## Novidade da v5.2 — você só preenche UMA variável

Antes era preciso preencher 5 variáveis na hospedagem, e bastava errar uma para
o deploy "virar" mas nada funcionar. Agora o servidor tem padrões inteligentes:

| Antes você digitava | Agora |
|---|---|
| `WPP_API_KEY` | já vem com padrão que combina com o painel |
| `WPP_AGENT_SECRET` | já vem com padrão que combina com a IA |
| `SUPABASE_URL` | já vem apontando para o Backend certo |
| `SUPABASE_ANON_KEY` | já vem com padrão |
| `WPP_AUTH_DIR` | **o servidor encontra o disco sozinho** |
| `WPP_PUBLIC_URL` | **detectada sozinha** |
| `SUPABASE_SERVICE_KEY` | **a única que você precisa informar** |

Ou seja: crie o serviço, cole a chave de serviço do banco e pronto. Se algo
falhar, abra `SUA_URL/api/diagnostico` e ele diz em português o que está faltando.

---

## 1. Qual hospedagem usar

| Plataforma | Custo mensal | Dorme? | Disco | Veredito |
|---|---|---|---|---|
| **Render (plano Starter)** | ~US$ 7,25 (≈ R$ 39) | Não | Sim (US$ 0,25/GB) | **Padrão do projeto.** Só cliques, sem terminal |
| Railway | US$ 5 (≈ R$ 27) | Não | Sim | Funciona igual, um pouco mais barato |
| VPS (Hetzner/Contabo) | €4–5 (≈ R$ 25–30) | Não | Sim | Mais barato no longo prazo, exige terminal |
| Oracle Cloud Free | R$ 0 | Não | Sim | Existe plano gratuito, cadastro chato e máquina lenta |
| Fly.io | ~US$ 3,84 (≈ R$ 21) | Não | Sim | Foi a origem dos erros de porta. **Não recomendado** |

**Escolha do projeto: Render, plano Starter.** Cumpre os três requisitos de
produção (processo sempre vivo, disco persistente, reinício automático) e é o
mais difícil de errar.

Sobre "conexão boa": o que pesa não é o país do servidor, e sim ele nunca dormir,
ter disco e ficar perto do Backend. Use a região **virginia** (EUA, costa leste),
a mais próxima do Brasil (~120 ms). A conversa com o WhatsApp é feita pelos
servidores do WhatsApp, não pelo Brasil.

Custos extras possíveis: disco de 1 GB (US$ 0,25 — a sessão ocupa poucos MB) e
tráfego de saída (volume ínfimo).

---

## 2. Publicar no Render (caminho padrão, ~10 minutos)

Tudo pelo navegador, sem terminal. Uma vez só.

1. **GitHub.** Suba esta pasta em um repositório novo (pode ser privado).
   `package.json`, `Dockerfile` e `render.yaml` precisam ficar na **raiz** do
   repositório, não dentro de uma subpasta.
2. **Criar o serviço.** render.com → **New** → **Blueprint** → conecte o GitHub
   → escolha o repositório. O Render lê o `render.yaml` e já monta tudo: imagem
   Docker, plano, disco e checagem de saúde.
3. **Revisar.** Confirme na tela de revisão:
   - plano **Starter** (o Free **não** serve)
   - disco montado em **`/var/data`**, 1 GB
   - região **virginia**
4. **Variáveis.** O Render vai pedir **apenas uma**:
   ```
   SUPABASE_SERVICE_KEY = <chave de serviço do Backend>
   ```
   Onde achar: painel do Backend → Configurações → chaves da API →
   `service_role`. `TZ` já vem preenchida pelo blueprint.
   **Não cadastre `PORT`** e não precisa cadastrar `WPP_AUTH_DIR`.
5. **Deploy.** O primeiro build leva de 3 a 8 minutos (instala o Baileys e
   compila dependências). Espere o status virar **Live**.
6. **Endereço.** Aparece no topo do serviço: `https://seu-servico.onrender.com`.
7. **Confira.** Abra `https://seu-servico.onrender.com/api/diagnostico`. Você
   deve ver `"ok": true`, `"pasta_persistente": true` e `"permissoes_banco":
   "completa"`. Depois cole só o endereço na aba **Conexão** do painel.

> Por que Starter e não Free: no Free o serviço hiberna após 15 min sem acesso e
> não tem disco. Hibernou = sessão caiu = QR Code de novo. Não existe contorno.

---

## 2.1 Alternativa: Railway (também funciona bem)

1. railway.app → **New Project** → **Deploy from GitHub repo** → escolha o
   repositório. Ele encontra o `Dockerfile` sozinho.
2. **Settings → Region:** `us-east4 (Virginia)`.
3. **Settings → Volumes → Add Volume:** montagem em **`/data`**.
4. **Variables:** só `SUPABASE_SERVICE_KEY`. Não cadastre `PORT`.
5. **Settings → Networking → Generate Domain.**
6. Abra `/api/diagnostico` e confirme `"ok": true`.

Diferença única: aqui o disco é `/data`; no Render é `/var/data`. Como o servidor
procura os dois sozinho, você não precisa configurar nada.

---

## 3. Publicar em VPS próprio

```bash
# na VPS, com Docker instalado
docker compose up -d --build
```

O `docker-compose.yml` já cria o volume e reinicia o serviço sozinho (a pasta da
sessão é detectada automaticamente). Para HTTPS, coloque um proxy (Caddy ou
Nginx) na frente apontando para a porta 8080.

---

## 4. Variáveis de ambiente

| Variável | Obrigatória | O que é |
|---|---|---|
| `SUPABASE_SERVICE_KEY` | **sim** | Chave de serviço do Backend. Sem ela o servidor conecta, mas o banco fica em modo limitado (cobrança/confirmação podem falhar). |
| `PORT` | não | Porta HTTP. Deixe em branco — a hospedagem injeta a sua. O servidor também escuta em 3000 e 3001 por segurança. |
| `WPP_API_KEY` | não | Senha que o painel usa para falar com este servidor. Tem padrão compatível. |
| `WPP_AGENT_SECRET` | não | Chave da função `wpp-agente` (IA). Tem padrão compatível. |
| `SUPABASE_URL` | não | Endereço do Backend. Tem padrão correto. |
| `SUPABASE_ANON_KEY` | não | Chave pública do Backend. Tem padrão. |
| `WPP_AUTH_DIR` | não | Pasta da sessão. **Detectada sozinha.** Só use para forçar um caminho. |
| `WPP_PUBLIC_URL` | não | URL pública. Detectada sozinha no Render/Railway/Fly. |
| `TZ` | recomendada | `America/Sao_Paulo` para os horários saírem certos. |

Nunca escreva as chaves em arquivos versionados. Use os "Secrets" da plataforma.

---

## 5. Como saber se está tudo bem

- `SUA_URL/api/diagnostico` → diz em português o que está faltando, se algo
  estiver. Este é o primeiro lugar para olhar quando algo não funciona.
- `SUA_URL/api/health` → resposta rápida usada pela hospedagem (deve ter
  `"ok":true`).
- Na aba **Conexão** do painel, o diagnóstico mostra versão, tempo no ar, sessão
  salva, contagem de mensagens e a data da última cobrança.
- Configure um monitor gratuito (UptimeRobot, BetterStack) apontando para
  `/api/health` a cada 5 minutos, com alerta por e-mail.

O que o servidor já garante sozinho:
- **Sem mensagem duplicada**: cada parcela é "reservada" no banco antes do envio.
- **Cobrança não se perde**: se o serviço estiver fora do ar às 11h, ele envia
  assim que voltar (e só uma vez por dia, mesmo reiniciando várias vezes).
- **Reconexão automática** com espera progressiva.
- **Fila do atendente humano** entregue em até 5 segundos.
- **Desligamento limpo**: em deploy/reinício ele fecha a sessão antes de morrer,
  evitando arquivo de credencial corrompido.

---

## 6. Backup e volta atrás

**Backup da sessão (opcional):** no Render, o disco pode ser baixado em
**Disks → seu disco**. No Railway, em **Settings → Volumes → Download**. No VPS,
basta copiar a pasta da sessão. Se perder a sessão, escaneie o QR Code
novamente — nenhum dado de cliente é perdido, porque tudo fica no Backend.

**Voltar para a hospedagem anterior:** este servidor não depende de nenhuma
plataforma específica. Publique esta mesma pasta no host antigo e cole a URL nova
na aba **Conexão**. Nada muda no código.

**Nunca apague o serviço antigo antes de o novo responder** em `/api/health`
e de o botão **Enviar teste** chegar no seu celular.

---

## 7. Problemas comuns

| Sintoma | Causa provável | Solução |
|---|---|---|
| Build falha no Render | `Dockerfile` fora da raiz ou build sem memória | Veja **Logs**; confira a raiz do repositório |
| "Live" mas a URL não responde | Disco não montado em `/var/data` | Confira em **Disks** e abra `/api/diagnostico` |
| Conecta e cai em minutos | Sem disco (sessão não persiste) | `/api/diagnostico` aponta `pasta_persistente: false` — crie o disco |
| Serviço "dormiu" | Plano Free | Mude para **Starter** |
| QR não aparece | Serviço ainda subindo | Espere o deploy ficar Live e clique em Conectar |
| Conectado mas não responde | Falta a chave da IA no Backend | Cadastre `OPENAI_API_KEY` no Backend |
| Cobrança não saiu hoje | Fora do horário ou sem conexão | Veja `last_cobranca` no diagnóstico do painel |

---

## 8. Portas (só se algum dia der "Request failed")

O servidor escuta em **8080, 3000 e 3001** ao mesmo tempo, e usa
`process.env.PORT` quando a plataforma injeta uma porta. Isso cobre Render,
Railway, Fly e VPS sem nenhuma configuração. Se a URL ficar muda com o serviço
ligado, confira se alguém cadastrou `PORT` manualmente com um valor errado e
remova essa variável.