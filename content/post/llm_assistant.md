---
title: "I built my own low-cost LLM assistant app"
summary: "Hosting my own chatbot on GCP using open-source LLM Gemma 3, Spring Boot and Angular."
date: 2025-07-21
tags: ["ai", "llm", "gemma", "spring", "spring boot", "java", "angular", "cloud run", "gcp", "firestore", "bruno"]
draft: true
---

*Source repository: [datatrigger/llm_app](https://github.com/datatrigger/llm_app/tree/main)*

Meet [Talian](assistant.vlg.engineer), my personal LLM assistant. It is a minimal ChatGPT clone giving me complete control over data privacy, deployed on GCP at low cost:

![Talian app](/res/llm_assistant/talian.png)

&nbsp;

## Architecture

![LLM app architecture](/res/llm_assistant/llm_app_architecture.png)

* Frontend: Angular 20 deployed on Netlify
* Backend: Spring Boot 3.5 REST API, deployed on GCP Cloud Run
* LLM Service: Gemma 3 4B model running on Ollama, deployed on GCP Cloud Run with GPU
* Database: Google Firestore for conversation persistence
* Authentication: GCP service-to-service authentication

The backend is stateless, so the state of the ongoing conversation must be maintained somehow. That is exactly the purpose of the document database (Firestore). Each message is stored with a conversation ID and timestamp:

![LLM app architecture](/res/llm_assistant/firestore.png)

Here the user prompt *What language would you advise for Advent of Code?* is part of the conversation *0jO7d3JznKvi179sbbtt*. This conversation ID is sent back to the frontend along with the LLM answer. If the user pursues the conversation, the new prompt will be sent to the backend along with the conversation ID. This way, the backend is able to fetch all the previous messages of the current conversation.

## Cost management

I am billed between $15 and $30 a month depending on my usage. Let's break it down.

#### Conversation data persistence

Initially, I was looking for a pay-per-use (sometimes called serverless pay-as-you-go) relational database. I found out there are such services outside of GCP (Supabase Postgres, AWS Aurora serverless v2) but on GCP's side, Cloud SQL incur charges even for idle instances. Since I wanted to stay on GCP, I went with document database Firestore instead.

Firestore is truly serverless, moreover, the free tier is very generous: tens of thousands of read/write requests and 1 Go of storage. More than enough for my application.

#### Backend

The Java Spring backend and the LLM server are both containerized and deployed on Cloud Run. I set the minimum number of active instances to 0, which means these containers only start when they receive a request.

Since the config for the Spring backend container is minimal (1 Go of RAM, 1 vCPU), the cost is low: $0.0000336 + $0.0000035 per second. That would be $3.20 a day if the instance ran 24/7. With the free tier and my actual usage, the actual cost is < 1$/month.

For the LLM server, I'm using a Nvidia L4 GPU at $0.0001867 per second. That would be $483 per month for an instance running 24/7... Now since I scale down to 0 instances when there is no request, the costs are manageable. Still, with my average billing is $0.5 to $1 a day, which mounts to $15-$30 a month. The big downside of scaling down to 0 is cold starts: I have to wait about 30s for my first prompt to be answered.

Cost: $15-$30/month

#### Frontend

It is hosted on Netlify at no cost, even with CI/CD (up to 5 hours of build time each month).

## Implementation Highlights

#### Frontend - Markdown integration

Since LLM responses often include formatted text or code blocks, I integrated the ngx-markdown library to properly render model responses:

```html
html@if (message.role === 'user') {
  {{ message.text }}
} @else {
  <markdown [data]="message.text"></markdown>
}
```

The CSS includes comprehensive styling for all markdown elements, from syntax-highlighted code to properly formatted tables and lists.

#### Frontend - Conversation persistence

Since the backend is stateless, the frontend has the responsibility to maintain the ongoing conversation object. Each conversation gets a unique ID from the backend, which the frontend stores and includes in subsequent requests:

```typescript
sendMessage(prompt: string, userId: string, conversationId?: string | null): Observable<PromptResponse> {
  const request: PromptRequest = {
    prompt,
    userId,
    ...(conversationId && { conversationId })
  };
  return this.http.post<PromptResponse>(`${this.apiUrl}/prompt`, request);
}
```

This design also prepares the ground for future enhancements, espcially conversation history and multi-device sync.

#### Backend - Architecture

The architecture follows the standard layered MVC pattern:

* Controller Layer
  * LlmController: the sole API endpoint that handles chat requests and responses

* Service Layer
  * LlmService: handles communication with my external LLM server
  * ConversationService: manages conversation persistence and retrieval from the database

* Data Layer
  * Message Entity: represents individual messages (text, role: user/model, timestamp)

* Configuration Layer:
  * Web config
  * Security config

* DTOs
  * LlmDto - Contains request/response models for both frontend communication and LLM API calls

#### Backend - Service-to-service authentication

Initially, I was using an API key to authenticate against the LLM server. Then I switched to service-to-service auth, allowing the backend to call the LLM server through IAM permissions.

```java
private String getIdTokenForCloudRun(String targetUrl) {
    GoogleCredentials credentials = GoogleCredential.getApplicationDefault();
    IdTokenProvider idTokenProvider = (IdTokenProvider) credentials;
    IdTokenCredentials idTokenCredentials = IdTokenCredentials
        .newBuilder()
        .setIdTokenProvider(idTokenProvider)
        .setTargetAudience(targetUrl)
        .build();
    return idTokenCredentials.getIdToken().getTokenValue();
}
```

The function `GoogleCredentials.getApplicationDefault()` is very convenient because it works in my dev setup too, as long as I have authenticated with the GCP CLI (gcloud).

#### Backend - Structured Logging with Context

The application uses structured JSON logging with MDC (Mapped Diagnostic Context) to add request-specific context to every log entry. The LlmController sets up tracing context at the start of each request:

```java
javaString requestId = UUID.randomUUID().toString().substring(0, 8);
MDC.put("requestId", requestId);
MDC.put("userId", request.userId());
```

Now the requestId and userId will be in every log entry pertaining to them.

## Backend - Testing External API Dependencies with WireMock

Testing the LLM service was tricky since it needs to make real HTTP calls to the LLM server.  I initially started with Mockito, but quickly realized that only the RestClient bean would be mocked. All the HTTP layers, response parsing, error handling, and serialization must still be written for every single test case. Instead, WireMock spins up an entire fake HTTP server that handles the whole request/response cycle. Setting it up is pretty straightforward with `@EnableWireMock` and `@ConfigureWireMock(baseUrlProperties = "llm.base.url")`. Now we can verify that our conversation history gets serialized correctly in the actual HTTP request:

```java
// Verify history was included in request
wireMockServer.verify(postRequestedFor(urlEqualTo(API_PATH))
    .withRequestBody(matchingJsonPath("$.contents[0].parts[0].text", equalTo("Previous user message")))
    .withRequestBody(matchingJsonPath("$.contents[1].role", equalTo("model")))
    .withRequestBody(matchingJsonPath("$.contents[2].parts[0].text", equalTo("Current message"))));
```

WireMock is also helpful to test various failure scenarios, i.e. fake network timeouts with `withFixedDelay(some_time)`, throw 500 errors, or return completely broken JSON. This way we know our error handling actually works in these cases.

#### Backend - Input validation

The backend leverages Spring's built-in validation framework through Jakarta Bean Validation annotations to ensure data integrity. This declarative approach provides clean, readable validation rules that are automatically enforced by the Spring container:

```java
@NotBlank(message = "Message text cannot be empty")
private String text;

@NotBlank(message = "Role cannot be empty")
@Pattern(regexp = "^(user|model)$", message = "Role must be either 'user' or 'model'")
private String role;
```

When validation fails, Spring automatically generates appropriate HTTP 400 (Bad Request) responses with detailed error messages, providing clear feedback to API clients while maintaining security by not exposing internal system details.

## What about the LLM server?

First, I went the Hugging Face/PyTorch route: import the `transformers` module, create a pipeline with whatever model you want, and that's it. This used to work seamlessly [back in the days](https://vlg.engineer/post/nlp_hugging_face/). But now, the models are so huge (I use [gemma-3-4b-it](https://huggingface.co/google/gemma-3-4b-it)) that going this way without further optimization requires unreasonable resources. When I rented a decent VM with a GPU and ran gemma 3 with tranformers and PyTorch, the model would take at least 30 seconds to answer enven the most simple yes-no prompt.

Next, I searched about ways to efficiently run these models and discoverd Ollama. It turns out Google already built an open-source docker image to run Gemma 3 with Ollama: [google-gemini/gemma-cookbook](https://github.com/google-gemini/gemma-cookbook/tree/main/Demos/Gemma-on-Cloudrun)

In the end, I just forked this repo and made a few adjustments for my use-case. Most notably, I removed the api key check in order to use service-to-service authentication. I made this change in the Golang proxy server that translates between Google's Gemini AI API format and OpenAI's API format.

## Continuous integration

The entire source code is in a single repo with a folder for each component: frontend, backend and llm-server. Each push on the main branch affecting a folder triggers the build/publish of the corresponding component. For the frontend, I use Netlify's built-in continuous deployment feature. For the backend and the llm server, I use GCP's Cloud Build.

#### Buildpacks



## Codespaces

## API testing with Bruno

## Next steps

Thanks for reading!
