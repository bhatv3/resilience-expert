# Skill: System Fingerprinting & Orchestration

## Objective
Identify the language, framework, and entry points to scope the analysis correctly.

## Instructions
1.  **Language Detection:**
    * **Java:** Look for `pom.xml`, `build.gradle`. Identify if Spring Boot, Quarkus, or Jakarta EE.
    * **Go:** Look for `go.mod`. Identify if using Gin, Echo, or standard `net/http`.
    * **Python:** Look for `requirements.txt`, `pyproject.toml`. Identify FastAPI, Flask, Django.

2.  **Entry Point Mapping:**
    * Scan for **Synchronous** inputs: `@RestController`, `http.HandleFunc`, `@app.route`.
    * Scan for **Asynchronous** inputs: `@KafkaListener`, `sarama.Consumer`, `sqs.ReceiveMessage`, `Celery`.
    * *Output:* Store this list in memory to pass to the `Lenses` skill.