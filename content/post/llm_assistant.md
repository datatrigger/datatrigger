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

# Architecture

![LLM app architecture](/res/llm_assistant/llm_app_architecture.png)

* Frontend: Angular 20 with signals (zoneless), deployed on Netlify
* Backend: Spring Boot 3.5 REST API, deployed on GCP Cloud Run
* LLM Service: Gemma 3 4B model running on Ollama, deployed on GCP Cloud Run with GPU
* Database: Google Firestore for conversation persistence
* Authentication: GCP service-to-service authentication

The backend is stateless, so the state of the ongoing conversation must be maintained somehow. That is exactly the purpose of the document database (Firestore). Each message is stored with a conversation ID and timestamp:

![LLM app architecture](/res/llm_assistant/firestore.png)

Here the user prompt *What language would you advise for Advent of Code?* is part of the conversation *0jO7d3JznKvi179sbbtt*. This conversation ID is sent back to the frontend along with the LLM answer. If the user pursues the conversation, the new prompt will be sent to the backend along with the conversation ID. This way, the backend is able to fetch all the previous messages of the current conversation.

# Cost management

## Conversation data persistence

Initially, I was looking for a pay-per-use (sometimes called serverless pay-as-you-go) relational database. I found out there are such services outside of GCP (Supabase Postgres, AWS Aurora serverless v2) but on GCP's side, Cloud SQL incur charges even for idle instances. Since I wanted to stay on GCP, I went with document database Firestore instead.

Firestore is truly serverless, moreover, the free tier is very generous: tens of thousands of read/write requests and 1 Go of storage. More than enough for my application.

# Backend

The Java Spring backend and the LLM server are both containerized and deployed on Cloud Run. I set the minimum number of active instances to 0, which means these containers only start when they receive a request.

Since the config for the Spring backend container is minimal (1 Go of RAM, 1 vCPU), the actual costs are negligeable (< $1) given my average usage.

For the LLM server, the costs are way higher.


# LLM server

*Forked from [google-gemini/gemma-cookbook](https://github.com/google-gemini/gemma-cookbook/tree/main/Demos/Gemma-on-Cloudrun)*
