---
title: "LLM draft"
summary: "Hosting my own chatbot on GCP using open-source LLM Gemma 3, Spring Boot and Angular. For privacy, and for fun of course."
date: 2025-07-21
tags: ["ai", "llm", "gemma", "spring", "spring boot", "java", "angular", "cloud run", "gcp", "firestore", "bruno"]
draft: true
---


* Final product
    * assistant.vlg.engineer
    * Screenshot
    * Features

* Architecture
    * Diagram with LLM, Spring Boot backend, GCP Firestore for persistence, frontend Angular.
    Service-to-service authentication backend - LLM

* LLM container
    * Need for a proxy
    * Container with Ollama

* Spring Boot backend
    * Architecture: controller, service, entity
    * Input validation
    * Authentication between backend and LLM container (during dev also )

* Frontend
    * Architecture
    * ...
    
* Development in codespaces
    * .devcontainer, post-create command for angular, bruno, gcloud...
    * secrets management
    * Docker in docker

* Testing
    * API: Bruno
    * backend: Spring Boot

* Cost-effective deployments
    * Cloud Run -> stateless applications
    * Persistence - > Firestore
    * CI/CD: cloud build
    * Service-to-service authentication

* Actual costs

* Missing
    * Authentication
    * User access to previous conversations
    * Streaming
    * Personas: I'd like to be able to select a "profile" associated with a prompt for context. E.g. if I'm working, I'd like the model to be aware of my projects and tools. If it's medical questions, then I'd like the model to know about my health conditions.