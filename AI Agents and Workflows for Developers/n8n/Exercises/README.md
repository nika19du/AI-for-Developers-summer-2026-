# n8n AI Agent – Workflow Notes

<img width="1640" height="781" alt="image" src="https://github.com/user-attachments/assets/07c1a7c2-d97a-4be7-a60f-798ddfb50cd8" />


## 1. Общ преглед

Всяко съобщение, което изпращам към Telegram бота, се прихваща от **Telegram Trigger** в n8n.

След това **Switch** node проверява какъв тип съдържание е получено:

* текст
* аудио
* изображение

и го насочва към съответния клон на workflow-а.

```text
Telegram
    ↓
Telegram Trigger
    ↓
Switch
   / | \
audio text image
```

---

## 2. Audio workflow

**Ако получим аудио**, първо изтегляме аудио файла от Telegram.

След това използваме **OpenAI transcription model**, за да преобразуваме речта в текст (**Speech-to-Text**).

```text
Telegram Audio
      ↓
Get Audio
      ↓
Transcribe Recording
      ↓
Text
      ↓
AI Agent
```

Полученият текст се подава към **AI Agent-а**.

Agent-ът използва своя **LLM (Large Language Model)** и решава дали има нужда да извика някой от предоставените му инструменти.

След като Agent-ът получи крайния отговор, можем:

* да го върнем като текст;
* или да го преобразуваме обратно в аудио чрез **audio generation / Text-to-Speech модел** и да го изпратим в Telegram.

```text
AI Agent
    ↓
   If
  /  \
Text  Audio
 ↓      ↓
Telegram Generate Audio
          ↓
       Telegram
```

---

## 3. Image workflow

**Ако получим изображение**, първо го изтегляме от Telegram като **binary data**.

След това подготвяме изображението и го подаваме към **vision-capable AI model**, който може да анализира изображения.

Ако към снимката има текст/caption с въпрос, моделът анализира изображението спрямо този въпрос.

Например:

> "Какво има на тази снимка?"

Ако няма въпрос, моделът може просто да опише или анализира съдържанието на изображението.

След това форматираме резултата и го изпращаме обратно в Telegram.

```text
Telegram Image
      ↓
Download Image
      ↓
Prepare Image
      ↓
Analyze Image
      ↓
Format Image Output
      ↓
Telegram
```

**Важно:** В текущия workflow този клон **не преминава през AI Agent-а**. Vision моделът анализира изображението директно.

---

## 4. Text workflow

**Ако получим текст**, подготвяме входа и го подаваме директно към **AI Agent-а**.

```text
Telegram Text
      ↓
Agent Input
      ↓
AI Agent
```

Agent-ът разполага с набор от инструменти (**tools**) и сам решава кои са необходими за конкретната заявка.

Например може:

* да прочете имейли;
* да изпрати имейл;
* да провери Google Calendar;
* да създаде Calendar event;
* да потърси информация в интернет;
* да потърси информация в Pinecone knowledge base.

---

# 5. Какво е LLM?

**LLM = Large Language Model (голям езиков модел).**

LLM е **вид AI модел**, предназначен основно за работа с език.

Примери са GPT, Claude, Gemini, Llama и други.

```text
Artificial Intelligence
        ↓
    AI Models
        ↓
       LLM
        ↓
 GPT / Claude / Gemini / Llama
```

Не всеки AI модел е LLM.

Например:

* Speech-to-Text модел → AI model
* Image generation модел → AI model
* Embedding модел → AI model
* LLM → AI model

---

# 6. LLM ≠ AI Agent

Това е важно разграничение.

**LLM** е моделът, който разбира текста, генерира отговори и може да взема решения относно следващото действие.

**AI Agent** е цялата система около LLM.

Опростено:

```text
AI Agent
 =
LLM
+
Instructions
+
Tools
+
Memory
+
Agent Loop
```

При моя workflow:

```text
                   ┌── Read Emails
                   ├── Send Email
                   ├── Get Calendar
                   ├── Create Calendar
User → AI Agent ───┼── Google Search
         ↕         └── Pinecone
        LLM
         ↕
       Memory
```

Gmail, Calendar, Search и Pinecone са **tools**.

Agent-ът е **оркестраторът**, който решава кой tool да използва.

---

# 7. Как AI Agent използва tools?

Tools не се изпълняват автоматично един след друг.

Например ако кажа:

> "Изпрати имейл на Иван от контактите ми, че срещата утре отпада."

Agent-ът може да изпълни:

```text
User request
     ↓
LLM
     ↓
"Не знам email адреса на Иван."
     ↓
Pinecone Search
     ↓
ivan@example.com
     ↓
LLM
     ↓
"Вече имам адреса. Трябва да изпратя email."
     ↓
Send Email
     ↓
Success
     ↓
LLM
     ↓
"Имейлът беше изпратен."
```

Тоест имаме **agent loop**:

```text
LLM
 ↓
Choose Tool
 ↓
Execute Tool
 ↓
Observe Result
 ↓
LLM
 ↓
Choose next action
 ↓
...
 ↓
Final Answer
```

Не сме програмирали предварително:

```text
Pinecone → Gmail
```

Дали Pinecone, Gmail, Calendar или друг инструмент ще бъде използван се определя динамично от Agent-а според заявката.

---

# 8. Pinecone Vector Store

**Pinecone Vector Store** служи като външна **knowledge base**.

В него можем да съхраняваме информация, която LLM по принцип няма как да знае, например:

* наши контакти;
* фирмена информация;
* документи;
* бележки;
* специфични за приложението данни.

При търсене използваме **embeddings**.

Опростено:

```text
"Намери контакта на Иван"
           ↓
Embedding Model
           ↓
[0.12, -0.45, 0.83, ...]
           ↓
Pinecone
           ↓
Semantic Similarity Search
           ↓
Най-релевантните данни
           ↓
AI Agent
```

Embedding моделът преобразува текста в числов вектор.

Pinecone сравнява този вектор с векторите в базата и намира **семантично най-близката информация**.

### Dimension

Размерността трябва да съвпада между embedding модела и Pinecone index-а.

Например:

```text
OpenAI Embeddings → 1536 dimensions
                         ↓
Pinecone Index → 1536 dimensions
```

Ако имаме:

```text
OpenAI → 1536
Pinecone → 512
```

ще получим:

```text
Vector dimension 1536 does not match
the dimension of the index 512
```

---

# 9. Simple Memory

**Simple Memory** пази контекста на разговора.

Например:

```text
User:
Казвам се Никол.

AI:
Приятно ми е!

...

User:
Как се казвам?

AI:
Никол.
```

LLM сам по себе си не трябва да се разглежда като постоянна памет.

Memory компонентът предоставя необходимата предишна информация обратно на Agent-а/модела.

---

# 10. n8n workflow vs AI Agent

Тук имаме **две нива на orchestration**.

### Ниво 1 – n8n

n8n управлява целия workflow:

```text
Telegram
    ↓
Switch
    ↓
Prepare Input
    ↓
AI Agent
    ↓
Format/Generate Output
    ↓
Telegram
```

### Ниво 2 – AI Agent

Agent-ът управлява инструментите:

```text
            ┌── Gmail
            ├── Calendar
LLM ↔ Agent ┼── Search
            ├── Pinecone
            └── Memory
```

Следователно:

**n8n orchestrates the workflow.**

**AI Agent orchestrates its available tools.**

---

# 11. Как се изпълнява n8n workflow?

n8n workflow-ът представлява **граф от зависимости**.

При последователност:

```text
A → B → C
```

`C` зависи от резултата на `B`, а `B` зависи от `A`.

Не трябва да мислим за workflow-а просто като за:

```text
изпълни абсолютно всичко едновременно
```

Nodes се изпълняват според връзките и зависимостите в workflow-а.

При **Switch** например изпълняваме клона, който отговаря на входните данни:

```text
              ┌→ Audio branch
Telegram → Switch → Text branch
              └→ Image branch
```

---

# 12. Защо използваме ngrok?

Моят n8n работи локално чрез Docker:

```text
http://localhost:5678
```

Проблемът е, че `localhost` е достъпен **само от моя компютър**.

Telegram сървърите не могат да направят:

```text
Telegram → http://localhost:5678
```

защото за Telegram `localhost` не означава моя компютър.

Освен това Telegram webhook изисква публично достъпен **HTTPS URL**.

Затова използваме **ngrok**.

Ngrok създава публичен HTTPS tunnel:

```text
Internet
   ↓
https://xxxxx.ngrok-free.dev
   ↓
ngrok tunnel
   ↓
http://localhost:5678
   ↓
Docker
   ↓
n8n
```

В моя случай архитектурата е:

```text
Telegram
    ↓
Telegram Servers
    ↓
HTTPS Webhook
    ↓
ngrok
    ↓
localhost:5678
    ↓
Docker Container
    ↓
n8n
```

---

# 13. WEBHOOK_URL в Docker

Тъй като n8n работи в Docker, задаваме публичния ngrok URL чрез environment variable в `docker-compose.yml`:

```yaml
services:
  n8n:
    image: n8nio/n8n:2.33.3
    container_name: softuni_ai_n8n
    ports:
      - 5678:5678
    environment:
      - WEBHOOK_URL=https://xxxxx.ngrok-free.dev/
    volumes:
      - n8n_data:/home/node/.n8n
```

След промяна на конфигурацията:

```powershell
docker compose down
docker compose up -d
```

n8n продължава да бъде достъпен локално на:

```text
http://localhost:5678
```

но webhook URL-ите, които предоставя на външни услуги, използват публичния ngrok адрес.

---

# 14. Кога ми трябва ngrok?

### n8n → Telegram

Например:

```text
n8n → Telegram → Send Message
```

Това е **outgoing request**.

n8n сам се свързва с Telegram API.

За това не е необходим ngrok.

### Telegram → n8n

Например:

```text
Telegram → Telegram Trigger → n8n
```

Това е **incoming request**.

Telegram трябва да може да достигне моя n8n отвън.

Понеже n8n работи на localhost, тук ни трябва публичен endpoint:

```text
Telegram
   ↓
ngrok HTTPS URL
   ↓
localhost:5678
   ↓
n8n
```

Затова **Telegram Trigger изисква ngrok или друг публично достъпен HTTPS адрес**, когато n8n се хоства локално.

---

# 15. Chat ID и ngrok решават различни проблеми

**Chat ID** отговаря на:

> На кого да изпратя Telegram съобщението?

```text
n8n
 ↓
Telegram API
 ↓
Chat ID
 ↓
User
```

**ngrok** отговаря на:

> Как Telegram да достигне моя локален n8n?

```text
Telegram
 ↓
ngrok
 ↓
localhost
 ↓
n8n
```

Следователно:

```text
Chat ID = WHO?
ngrok   = HOW TO REACH MY LOCAL SERVER?
```

---

# 16. Най-краткото обобщение

Цялата система може да се мисли така:

```text
USER
 ↓
TELEGRAM
 ↓
NGROK
 ↓
N8N
 ↓
SWITCH
 ├── TEXT ───────────────┐
 │                       ↓
 ├── AUDIO → STT ───→ AI AGENT
 │                       ↓
 └── IMAGE → VISION    LLM
                       ↕
                 ┌──── Tools ────┐
                 │ Gmail         │
                 │ Calendar      │
                 │ Search        │
                 │ Pinecone      │
                 │ Memory        │
                 └───────────────┘
                       ↓
                    RESPONSE
                       ↓
              Text or generated audio
                       ↓
                    TELEGRAM
```

### Ключови понятия

**n8n** → orchestration на целия workflow <br/>
**Telegram Trigger** → стартира workflow при нов Telegram update <br/>
**Switch** → определя кой клон да бъде изпълнен <br/>
**LLM** → голям езиков AI модел <br/>
**AI Agent** → LLM + instructions + tools + memory + agent loop <br/>
**Tools** → външни възможности на Agent-а <br/>
**Memory** → контекст от разговора <br/>
**Embeddings** → числово представяне на семантично съдържание <br/>
**Pinecone** → vector database / knowledge base <br/>
**RAG** → намиране на външна информация и предоставянето ѝ на модела <br/>
**ngrok** → публичен HTTPS tunnel към локалния n8n <br/>
**Webhook** → начин външна система да извести n8n за настъпило събитие <br/>
