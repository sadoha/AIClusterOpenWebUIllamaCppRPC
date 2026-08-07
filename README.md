# Author: Andrii Sadovskyi <andrii.sadovskyi@gmail.com>
# AI Cluster on Apple Silicon ARM64: llama.cpp RPC + Open WebUI + Hermes Agent

<p align="center">
  <img src="logo.jpeg" alt="Project logo"/>
</p>

This project deploys a local AI stack for Apple Silicon using Docker Compose. It includes:

- `llama.cpp` distributed inference (`llama-server` head + two `ggml-rpc-server` workers)
- `Open WebUI` as the human-facing interface
- `Hermes Agent` as a configurable agent gateway connected to the same local model endpoint

## Prerequisites

- Docker
- Docker Compose
- Git

## Usage

1.  **Configure Environment**

    Copy the example environment file and update it with your secrets and preferences.

    ```bash
    cp .env.example .env
    # Edit .env with your values (HF_TOKEN, etc.)
    ```

2.  **Start the Stack**

    Build and run all services in detached mode.

    ```bash
    docker compose up -d --build
    ```

3.  **Access Services**

    -   **Open WebUI:** http://127.0.0.1:3000
    -   **Hermes Dashboard:** http://127.0.0.1:9119

4.  **Stop the Stack**

    ```bash
    docker compose down
    ```

## Official Documentation References

-   Docker Compose File
-   Docker Environment Variables
-   Docker Security
-   Docker Multi-Platform Builds
-   llama.cpp Server
-   Open WebUI Docs
-   Hermes Agent Docker Guide
