# API Integration Guide — Android (Kotlin)

How to talk to the **chatgpt-chatbot** backend from an Android app.

The server wraps the OpenAI Responses API with web search. You send a message, you get a reply plus the sources it cited. Multi-turn conversations are handled by the server — you never send chat history, only a conversation id.

---

## 1. The essentials

| | |
|---|---|
| **Base URL** | `https://chatbot.funspark.uk/` |
| **Auth** | `x-api-key: <key>` header on every `/api/*` request |
| **Content type** | `application/json` |
| **Transport** | Always use `https://`. The key travels in a header, so plain HTTP exposes it. |

The server is already deployed and reachable — nothing to run locally. Verify it right now, no key needed:

```bash
curl https://chatbot.funspark.uk/health
# {"ok":true,"model":"gpt-4o-mini","conversations":0}
```

> **Use `https://` explicitly.** The host currently answers on plain HTTP too, without redirecting, so a `http://` URL would send your API key in cleartext over the network. Hard-code the scheme; never build the URL from user input or config that could downgrade it.

### Endpoints

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/chat` | Send a message, get the whole reply at once |
| `POST` | `/api/chat/stream` | Same, streamed token by token (SSE) |
| `DELETE` | `/api/conversations/{id}` | Delete a conversation |
| `GET` | `/health` | Liveness check — **no API key needed** |

---

## 2. Two rules that will bite you

Read these before writing code. They cause almost all integration bugs with this API.

**① Requests are slow. Set a long read timeout.**

A reply that triggers a web search routinely takes 10–60 seconds. OkHttp's default read timeout is 10 seconds, so with default settings **most real requests fail**. Use 180 seconds, matching the server's nginx timeout.

**② A conversation id can stop working at any time.**

The server keeps conversations in memory. When it restarts — a deploy, a crash, a reboot — every existing id is forgotten and returns **404**. Your app must treat 404 as "this chat is gone, start a new one", not as a fatal error. Never assume an id survives forever.

---

## 3. Setup

### Dependencies

```kotlin
// app/build.gradle.kts
plugins {
    kotlin("plugin.serialization") version "2.0.21"
}

dependencies {
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
    implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:1.0.0")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.9.0")
}
```

### Manifest

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

### Base URL

```kotlin
// Trailing slash is required by Retrofit.
const val BASE_URL = "https://chatbot.funspark.uk/"
```

Because the API is HTTPS, you need **no** `networkSecurityConfig` and **no** `usesCleartextTraffic` — Android's defaults already permit it. If you ever find yourself adding `usesCleartextTraffic="true"` to make this work, something is wrong with the URL, not with Android.

---

## 4. Models

```kotlin
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

@Serializable
data class ChatRequest(
    val message: String,
    val conversationId: String? = null,   // omit to start a new conversation
)

@Serializable
data class ChatResponse(
    val conversationId: String,
    val reply: String,
    val sources: List<Source> = emptyList(),
)

@Serializable
data class Source(
    val url: String,
    val title: String,
)

@Serializable
data class ApiError(val error: String)

@Serializable
data class Health(
    val ok: Boolean,
    val model: String,
    val conversations: Int,
)

@Serializable
data class DeleteResponse(val ok: Boolean)
```

`sources` is often empty, and that is not a bug. Web search is a tool the model *chooses* to use. "What year did WW2 end?" needs no search and cites nothing; "what's the World Cup schedule?" searches and returns a dozen sources.

---

## 5. The client

```kotlin
import okhttp3.Interceptor
import okhttp3.OkHttpClient
import java.util.concurrent.TimeUnit

class ApiKeyInterceptor(private val apiKey: String) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): okhttp3.Response {
        val request = chain.request().newBuilder()
            .addHeader("x-api-key", apiKey)
            .build()
        return chain.proceed(request)
    }
}

object Network {
    // Long read timeout: a web-search answer can take a minute. The 10s default
    // would fail most real requests.
    val client: OkHttpClient = OkHttpClient.Builder()
        .addInterceptor(ApiKeyInterceptor(BuildConfig.CHAT_API_KEY))
        .connectTimeout(15, TimeUnit.SECONDS)
        .readTimeout(180, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        // Streaming must not be capped by a total-call deadline.
        .callTimeout(0, TimeUnit.MILLISECONDS)
        .retryOnConnectionFailure(true)
        .build()
}
```

```kotlin
import retrofit2.Retrofit
import retrofit2.http.*
import kotlinx.serialization.json.Json
import com.jakewharton.retrofit2.converter.kotlinx.serialization.asConverterFactory
import okhttp3.MediaType.Companion.toMediaType

interface ChatApi {
    @POST("api/chat")
    suspend fun chat(@Body body: ChatRequest): ChatResponse

    @DELETE("api/conversations/{id}")
    suspend fun deleteConversation(@Path("id") id: String): DeleteResponse

    @GET("health")
    suspend fun health(): Health
}

object ApiFactory {
    private val json = Json { ignoreUnknownKeys = true }

    fun create(baseUrl: String): ChatApi = Retrofit.Builder()
        .baseUrl(baseUrl)   // e.g. "https://chatbot.funspark.uk/" — must end with "/"
        .client(Network.client)
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()
        .create(ChatApi::class.java)
}
```

`ignoreUnknownKeys = true` matters: it lets the server add response fields later without breaking already-shipped app versions.

---

## 6. Sending a message

### Request

```json
{ "message": "what is the fifa world cup 2026 schedule?" }
```

Include `conversationId` to continue an existing chat; omit it to start a new one.

### Response — `200`

```json
{
  "conversationId": "HToAUPx-Cu3fAPWy1EA5OizflvfVwwsQqlnuzt5GDYg",
  "reply": "The 2026 FIFA World Cup begins on Thursday, June 11, 2026…",
  "sources": [
    { "url": "https://inside.fifa.com/…", "title": "FIFA World Cup 26" }
  ]
}
```

**Always store the returned `conversationId` and send it on the next message.** That is the entire mechanism for multi-turn chat — you never send previous messages yourself.

### Worked example: a two-turn conversation

This is the whole protocol. Run these two commands against the live server and you will see it work.

**Turn 1 — no `conversationId`, so a new chat starts:**

```bash
curl --location 'https://chatbot.funspark.uk/api/chat' \
--header 'Content-Type: application/json' \
--header 'x-api-key: YOUR_KEY' \
--data '{
  "message": "Name the three host countries of the 2026 World Cup."
}'
```

```json
{
  "conversationId": "PqUq_h0p5ypAVw_74XH6KkVf-yIgAJEcJ1ZWC2IOMGs",
  "reply": "The three host countries for the 2026 FIFA World Cup are the United States, Canada, and Mexico.",
  "sources": []
}
```

**Turn 2 — pass that `conversationId` back:**

```bash
curl --location 'https://chatbot.funspark.uk/api/chat' \
--header 'Content-Type: application/json' \
--header 'x-api-key: YOUR_KEY' \
--data '{
  "message": "Which of those is the largest by area?",
  "conversationId": "PqUq_h0p5ypAVw_74XH6KkVf-yIgAJEcJ1ZWC2IOMGs"
}'
```

```json
{
  "conversationId": "PqUq_h0p5ypAVw_74XH6KkVf-yIgAJEcJ1ZWC2IOMGs",
  "reply": "The largest by area among the three host countries is the United States. It covers about 9.8 million square kilometers…",
  "sources": []
}
```

Notice what turn 2 does **not** contain: the previous question, the previous answer, any history at all. Just the new message and the id. The phrase *"which of those"* only resolves because the server pulled the earlier turn from the conversation — that is the whole point of `conversationId`.

Get this wrong and the symptom is unmistakable: drop the id and the model replies *"Which of what?"*, because every message starts a brand-new chat.

The same `conversationId` works across both endpoints — start a conversation on `/api/chat` and continue it on `/api/chat/stream`, or the reverse.

```kotlin
class ChatRepository(private val api: ChatApi) {

    private var conversationId: String? = null

    suspend fun send(message: String): Result<ChatResponse> = runCatching {
        api.chat(ChatRequest(message, conversationId))
    }.onSuccess {
        conversationId = it.conversationId
    }.recoverCatching { error ->
        // The server restarted and forgot this conversation. Retry once as a
        // new chat rather than surfacing an error the user cannot act on.
        if (error is retrofit2.HttpException && error.code() == 404) {
            conversationId = null
            api.chat(ChatRequest(message, null)).also { conversationId = it.conversationId }
        } else {
            throw error
        }
    }

    fun startNewChat() { conversationId = null }
}
```

That 404 recovery is the single most important piece of error handling in this guide. Without it, users hit a dead chat after every server deploy.

---

## 7. Streaming (`/api/chat/stream`)

Streaming shows text as it is generated instead of waiting a minute for a wall of text. Retrofit cannot model Server-Sent Events, so read the raw OkHttp response.

### Wire format

Frames are `event:` + `data:` pairs separated by a blank line:

```
event: start
data: {"conversationId":"1fIHqYomrnqtQiRAuCnF3vZsIJYShUGPk1I-TCp3MWg"}

event: status
data: {"text":"searching the web…"}

event: delta
data: {"text":"Hello"}

event: done
data: {"conversationId":"1fIH…","sources":[{"url":"…","title":"…"}]}
```

| Event | Payload | Meaning |
|---|---|---|
| `start` | `{ conversationId }` | Save this id immediately — it arrives before any text |
| `status` | `{ text }` | Progress note, e.g. "searching the web…" — show as a spinner label |
| `delta` | `{ text }` | A chunk of the answer. **Append** it; do not replace |
| `done` | `{ conversationId, sources }` | Finished; sources are only available here |
| `error` | `{ message }` | Generation failed mid-stream |

### Implementation

```kotlin
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.flow.flowOn
import kotlinx.serialization.builtins.ListSerializer
import kotlinx.serialization.json.*
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.Request
import okhttp3.RequestBody.Companion.toRequestBody

sealed interface ChatEvent {
    data class Started(val conversationId: String) : ChatEvent
    data class Status(val text: String) : ChatEvent
    data class Delta(val text: String) : ChatEvent
    data class Done(val conversationId: String, val sources: List<Source>) : ChatEvent
    data class Failed(val message: String) : ChatEvent
}

class ChatStream(
    private val baseUrl: String,
    private val json: Json = Json { ignoreUnknownKeys = true },
) {
    fun send(message: String, conversationId: String? = null): Flow<ChatEvent> = flow {
        val payload = json.encodeToString(ChatRequest.serializer(), ChatRequest(message, conversationId))

        val request = Request.Builder()
            .url("${baseUrl.trimEnd('/')}/api/chat/stream")
            .addHeader("Accept", "text/event-stream")
            .post(payload.toRequestBody("application/json".toMediaType()))
            .build()

        Network.client.newCall(request).execute().use { response ->
            if (!response.isSuccessful) {
                val body = response.body?.string().orEmpty()
                val detail = runCatching { json.decodeFromString<ApiError>(body).error }
                    .getOrElse { "HTTP ${response.code}" }
                emit(ChatEvent.Failed(detail))
                return@flow
            }

            val source = response.body?.source() ?: return@flow
            var eventName: String? = null

            // Read frame by frame. A blank line terminates a frame; the parser
            // must not assume event/data arrive in a single read.
            // readUtf8Line() returns null at end of stream — readUtf8LineStrict()
            // would throw EOFException if the last frame lacks a trailing newline.
            while (true) {
                val line = source.readUtf8Line() ?: break

                when {
                    line.startsWith("event: ") -> eventName = line.removePrefix("event: ").trim()

                    line.startsWith("data: ") -> {
                        val data = json.parseToJsonElement(line.removePrefix("data: ")).jsonObject
                        val event = when (eventName) {
                            "start" -> ChatEvent.Started(data["conversationId"]!!.jsonPrimitive.content)
                            "status" -> ChatEvent.Status(data["text"]!!.jsonPrimitive.content)
                            "delta" -> ChatEvent.Delta(data["text"]!!.jsonPrimitive.content)
                            "done" -> ChatEvent.Done(
                                conversationId = data["conversationId"]!!.jsonPrimitive.content,
                                sources = data["sources"]?.let {
                                    json.decodeFromJsonElement(ListSerializer(Source.serializer()), it)
                                } ?: emptyList(),
                            )
                            "error" -> ChatEvent.Failed(data["message"]!!.jsonPrimitive.content)
                            else -> null
                        }
                        event?.let { emit(it) }
                    }

                    line.isBlank() -> eventName = null   // end of frame
                }
            }
        }
    }.flowOn(Dispatchers.IO)
}
```

Collecting it:

```kotlin
val builder = StringBuilder()

chatStream.send(text, conversationId).collect { event ->
    when (event) {
        is ChatEvent.Started -> conversationId = event.conversationId
        is ChatEvent.Status  -> statusLabel.value = event.text
        is ChatEvent.Delta   -> { builder.append(event.text); replyText.value = builder.toString() }
        is ChatEvent.Done    -> { conversationId = event.conversationId; sources.value = event.sources }
        is ChatEvent.Failed  -> showError(event.message)
    }
}
```

**Cancelling the Flow stops the work.** When the user leaves the screen and the coroutine is cancelled, the connection closes, and the server aborts its OpenAI request — so you stop paying for text nobody will read. Collect from `viewModelScope` and this happens for free.

---

## 8. Errors

Every failure returns `{ "error": "..." }` with a meaningful status code.

| Code | Meaning | What the app should do |
|---|---|---|
| `400` | Bad input — empty message, over 4000 chars, malformed id | Fix before sending; validate client-side |
| `401` | Missing or wrong API key | Not user-recoverable. Log it; the build is misconfigured |
| `404` | Unknown conversation id (usually a server restart) | **Clear the id and retry as a new chat** |
| `413` | Request body over 256 KB | Shorten the message |
| `429` | Rate limit hit | Back off — honour the `Retry-After` header (seconds) |
| `502` | OpenAI unavailable | Transient. Offer a retry button |

```kotlin
suspend fun sendWithBackoff(message: String): ChatResponse {
    repeat(3) { attempt ->
        try {
            return api.chat(ChatRequest(message, conversationId))
        } catch (e: retrofit2.HttpException) {
            when (e.code()) {
                429 -> {
                    // The server tells you exactly how long to wait; don't guess.
                    val retryAfter = e.response()?.headers()?.get("Retry-After")?.toLongOrNull() ?: 5
                    kotlinx.coroutines.delay(retryAfter * 1000)
                }
                502 -> kotlinx.coroutines.delay((1L shl attempt) * 1000)
                else -> throw e
            }
        }
    }
    throw IllegalStateException("Failed after 3 attempts")
}
```

Never retry `400` or `401` — they fail identically every time.

---

## 9. Limits

| Limit | Default | Response when exceeded |
|---|---|---|
| Message length | 4000 characters | `400` |
| Request body | 256 KB | `413` |
| Requests per minute | 20 per API key | `429` |
| Requests per day | 200 per API key | `429` |
| Conversation idle life | 24 hours | `404` on next use |

Enforce the 4000-character limit in your UI so users get instant feedback instead of a round trip. Note the rate limits are **per API key, not per user** — every install sharing one key shares one allowance, so 20/minute is 20 across your whole user base.

---

## 10. Security

**The API key will be extracted from your APK.** Anyone can unzip an APK and read its strings, so treat this as a fact of life rather than something to solve. Do what you can, and design for leakage:

- Keep the key out of source control. Put it in `local.properties` and expose it via `BuildConfig`:

  ```kotlin
  // app/build.gradle.kts
  val chatApiKey: String = project.findProperty("CHAT_API_KEY") as String? ?: ""
  android.buildTypes.all {
      buildConfigField("String", "CHAT_API_KEY", "\"$chatApiKey\"")
  }
  ```

- Use a **separate key for Android** from web/iOS. If it leaks, the server operator revokes just that one — `API_KEYS` is a comma-separated list and rotation needs no downtime.
- **HTTPS only** in production, so the key isn't readable on the wire.
- Consider certificate pinning to make traffic interception meaningfully harder.
- Never log the key or the full request headers in a release build. Gate the OkHttp logging interceptor behind `BuildConfig.DEBUG`.

For real per-user security you need per-user identity — a login issuing a short-lived token — rather than one shared key. Ask the backend team; it's a server-side change.

---

## 11. Checklist

Before your first request:

- [ ] `INTERNET` permission in the manifest
- [ ] Base URL ends with `/`
- [ ] `x-api-key` header on every `/api/*` call
- [ ] Read timeout raised to 180s
- [ ] `404` clears the conversation id and starts a new chat
- [ ] `429` honours `Retry-After`
- [ ] `conversationId` stored and sent on follow-up messages

Verify the server is reachable before debugging your client — this needs no key:

```bash
curl https://chatbot.funspark.uk/health
# {"ok":true,"model":"gpt-4o-mini","conversations":2}
```

Then a full round trip:

```bash
curl -X POST https://chatbot.funspark.uk/api/chat \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_KEY" \
  -d '{"message":"hello"}'
```

If curl works and the app doesn't, the problem is in the client — check the timeout and the header first.
