# 🌤️ Chatbot de Clima no Telegram com n8n

Chatbot para Telegram, construído em **n8n**, que informa a **temperatura atual** de qualquer cidade do Brasil.
O usuário envia o nome da cidade (`Cidade,UF,BR`), o workflow normaliza o texto, consulta a **API gratuita do OpenWeather**, extrai e arredonda a temperatura e devolve uma mensagem curta e amigável.

> Exemplo de resposta: `🌤️ A temperatura em Belo Horizonte é de 25°C.`
> Em caso de erro: `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).`

---

## 🧩 Como o workflow funciona

| # | Nó | Tipo | O que faz |
|---|----|------|-----------|
| 1 | **Telegram Trigger** | `telegramTrigger` | Recebe as mensagens de texto enviadas ao bot (`updates: message`). |
| 2 | **Formatar entrada (queue)** | `Set` | Cria a variável **`queue`** com o texto tratado: remove espaços extras, remove acentuação (`normalize("NFD")`), converte para minúsculas, normaliza as vírgulas e garante o sufixo `,br`. Também guarda `chatId` e `textoOriginal`. |
| 3 | **OpenWeather** | `HTTP Request` | `GET https://api.openweathermap.org/data/2.5/weather` com **Send Query Parameters** ativo: `queue`, `units=metric`, `lang=pt_br` e `appid={{ $env.OPENWEATHER_API_KEY }}`. Usa *Full Response* + *Never Error* para que o status HTTP possa ser avaliado pelo IF. |
| 4 | **Resposta valida?** | `IF` | Valida `statusCode = 200`, a existência de `body.main.temp` e de `body.name`. Saída **true** → sucesso; saída **false** → mensagem de erro. |
| 5 | **Montar mensagem (fallback)** | `Code` | **Fallback determinístico.** Extrai `name`, `main.temp`, `main.feels_like` e a descrição; arredonda a temperatura com `Math.round()` e monta a string final. Sempre executa. |
| 6 | **Melhorar mensagem (Gemini)** | `Google Gemini` *(opcional, desativado)* | Reescreve a mensagem com tom mais natural. `temperature: 0.1`, saída em JSON `{"message":"...","ok":true}`. |
| 7 | **Mensagem final** | `Code` | Usa a saída do Gemini se ela existir e for válida; caso contrário mantém a mensagem do fallback. Marca a origem em `origem`. |
| 8 | **Enviar temperatura** | `Telegram` | Envia a mensagem de sucesso para o `chat.id` de origem. |
| 9 | **Enviar erro** | `Telegram` | Envia a mensagem de cidade não encontrada. |

```
Telegram Trigger → Formatar entrada (queue) → OpenWeather → Resposta valida?
                                                             ├── true  → Montar mensagem (fallback) → Melhorar mensagem (Gemini) → Mensagem final → Enviar temperatura
                                                             └── false → Enviar erro
```

> **Sobre o parâmetro `queue`:** o enunciado pede o parâmetro `queue` na query. Como o endpoint do OpenWeather espera o parâmetro `q`, o nó envia **os dois** (`q` e `queue`) com o mesmo valor — assim o requisito é atendido e a chamada funciona de verdade.

---

## 🔑 Variáveis de ambiente esperadas

| Variável | Onde é usada | Descrição |
|----------|--------------|-----------|
| `OPENWEATHER_API_KEY` | Nó **OpenWeather**, via `{{ $env.OPENWEATHER_API_KEY }}` | Chave da API do OpenWeather (https://home.openweathermap.org/api_keys). |
| `TELEGRAM_BOT_TOKEN` | Credencial **Telegram API** do n8n | Token do bot gerado pelo `@BotFather`. |

⚠️ **Nenhum token real está neste repositório.** O JSON do workflow referencia apenas a variável de ambiente e o nome da credencial.

---

## 🚀 Como importar e configurar

### 1. Importar o workflow

1. Abra o n8n → **Workflows** → botão **`...`** (canto superior direito) → **Import from File...**
2. Selecione o arquivo `workflow-chatbot-telegram.json`.
3. Clique em **Save**.

### 2. Configurar a variável `OPENWEATHER_API_KEY`

A chave é lida do **ambiente do n8n**, não de dentro do workflow.

**Docker / docker-compose:** adicione ao serviço do n8n e reinicie o container:

```yaml
environment:
  - OPENWEATHER_API_KEY=sua_chave_do_openweather
```

```bash
docker compose up -d --force-recreate
```

**Execução local via npm/npx:**

```bash
export OPENWEATHER_API_KEY=sua_chave_do_openweather
n8n start
```

> A variável só passa a existir para o n8n **após reiniciar o processo/container**.

### 3. Configurar a credencial do Telegram

1. No n8n, vá em **Credentials** → **Create Credential** → **Telegram API**.
2. Cole em **Access Token** o token do bot fornecido pelo `@BotFather` (valor de `TELEGRAM_BOT_TOKEN`).
3. Salve com um nome, por exemplo `Telegram Bot`.
4. Abra o workflow e selecione essa credencial nos três nós do Telegram: **Telegram Trigger**, **Enviar temperatura** e **Enviar erro**.

### 4. (Opcional) Ativar o Google Gemini

O nó **Melhorar mensagem (Gemini)** vem **desativado** de propósito, para que o workflow funcione e possa ser avaliado sem custo e sem credenciais de IA — nesse caso a mensagem vem do nó de **fallback determinístico**.

Para ativar:

1. **Credentials** → **Create Credential** → **Google Gemini (PaLM) API** e informe sua API key do Google AI Studio.
2. Abra o nó **Melhorar mensagem (Gemini)**, selecione a credencial e confirme o modelo (`models/gemini-2.5-flash`).
3. Clique com o botão direito no nó → **Activate** (ou pressione `D`) para reativá-lo.

O nó está posicionado **entre** `Montar mensagem (fallback)` e `Mensagem final`, e está configurado com `onError: continueRegularOutput`: se a chamada ao Gemini falhar, o fluxo continua e a mensagem do fallback é usada.

### 5. Ativar o bot

Clique em **Active** no canto superior direito do workflow. O n8n registra o webhook no Telegram automaticamente.

---

## 🧪 Como testar

Abra uma conversa com o seu bot no Telegram e envie:

| Mensagem enviada | Resposta esperada |
|------------------|-------------------|
| `Belo Horizonte,MG,BR` | `🌤️ A temperatura em Belo Horizonte é de 25°C.` |
| `São Paulo,SP` | `🌤️ A temperatura em São Paulo é de 22°C.` |
| `Recife` | `🌤️ A temperatura em Recife é de 30°C.` |
| `Cidadeinexistente123` | `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).` |

(Os valores de temperatura variam conforme o momento da consulta.)

Para testar dentro do n8n sem usar o Telegram, use **Test workflow** com um *pin data* no `Telegram Trigger`:

```json
{ "message": { "text": "Belo Horizonte,MG,BR", "chat": { "id": 123456 } } }
```

---

## ✅ Checklist da entrega

- [x] Trigger inicial com **Telegram Trigger** recebendo mensagens de texto
- [x] Variável **`queue`** criada em um nó **Set**, com trim, remoção de acentos e minúsculas
- [x] **HTTP Request** para `https://api.openweathermap.org/data/2.5/weather` com *Send Query Parameters* (`queue`, `units`, `lang`, `appid` via `$env`)
- [x] Extração e arredondamento da temperatura + string final formatada
- [x] **IF node** validando status HTTP e campos esperados, com desvio para a mensagem de erro
- [x] **Telegram Send Message** para sucesso e para erro
- [x] Nó **Google Gemini** (opcional) com **fallback determinístico** em nó Code
- [x] JSON exportado **sem credenciais ou tokens embutidos**

---

## 📦 Estrutura do repositório

```
.
├── workflow-chatbot-telegram.json   # workflow exportado do n8n
├── docker-compose.yml               # ambiente local do n8n (opcional)
├── .env.example                     # modelo das variáveis de ambiente
└── README.md
```
