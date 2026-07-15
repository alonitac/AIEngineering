# Final Project

## Overview

In this project you'll design and build an **AI agent** that solves a real-world problem.

Your agent will be a fully autonomous system - it reasons, plans, and acts using tools - deployed on a production-grade Kubernetes cluster with observability, CI/CD, and testing.

This project brings together everything you've learned in the course: agents, MCP, containers, Kubernetes, GitHub Actions, and observability.

## Requirements

### 1. Agent Design

Build an intelligent agent that:

- Solves a **clearly defined, real business problem** with measurable value
- Uses an LLM framework of your choice - **LangGraph**, LangChain, or equivalent. No no-code platforms (n8n, CrewAI, etc.) - you write the agent code
- Exposes an **HTTP API** at minimum; a web UI is highly recommended
- Well-crafted: retries, graceful termination, fallbacks, clear error responses to the user
- Has a system prompt that defines the agent's persona, capabilities, and boundaries

### 2. MCP Tooling

Your agent must use the **MCP** for tool integration:

- Connect to at least one publicly or self-hosted MCP server. Examples: GitHub MCP, Brave Search MCP, Slack MCP, Filesystem MCP, Kubernetes MCP, Gmail MCP
- You are highly encouraged to build your own MCP server that exposes tools specific to your domain. 

Your agent must call tools during a real interaction.

### 3. Kubernetes Deployment

Deploy your full stack to a Kubernetes cluster in both **`dev`** and **`prod`** namespaces:

- Use separate configuration for each env
- Well-crafted: liveness/readiness probes, resource requests/limits, HPA, secrets management, config maps, etc.
- Place all Kubernetes manifests in your repository

### 4. CI/CD Pipeline

Configure a **GitHub Actions** pipeline that:

- Runs all tests on every **pull request**
- Deploys to the **`dev`** namespace on merge to the `dev` branch
- Deploys to the **`prod`** namespace on merge to `main`
- Reports test results clearly (Allure, GitHub Actions summary, Codecov, or your tool of choice)

### 5. Observability

Instrument your system and make it visible:

- Expose **Prometheus metrics** from all services (at minimum: your agent and your local MCP server)
- Build useful **Grafana dashboards**

## Testing

Write automated tests for your project. Provide a clear **test plan** document (what you test, how, and your success criteria). Your tests must include:

- **Unit tests** - test agent logic and MCP server tools in isolation. Mock the LLM and external services
- **Integration tests** - test the interaction between your agent and your local MCP server using the real MCP transport

## Extra

- **Web UI** - a chat interface, dashboard, or any UI that makes your agent accessible to non-technical users
- **Multi-agent** - decompose your system into specialized sub-agents that collaborate to solve complex tasks

## Presentation

- Prepare a **20-minute** presentation with **slides** and a **live demo**
- Your slides must include:
  - A short introduction of yourself
  - Your problem statement and why it has real business value
  - Your architecture: agent, MCP servers, K8s, observability
  - Testing overview: what you tested, your strategy, success criteria
- Show a **live demo** of your agent handling a real request end-to-end
- Show a **Grafana dashboard** live, with real metrics from your demo
- Show your **CI pipeline** running in GitHub Actions
- Be ready to explain every decision you made - architecture, tools, testing, deployment
- **Bonus:** present in English


## Inspiring Use Cases

The following ideas are meant to inspire you. You are free to implement any of them or invent your own.


### DevOps Incident Response Agent

**Problem:** On-call engineers spend too much time manually correlating logs, metrics, and alerts to diagnose production incidents - often at 3 AM.

**What the agent does:** Receives a Prometheus alert, fetches the relevant pod logs, queries recent metrics, identifies the likely root cause, and suggests a fix as a PR, or opens a GitHub issue or pages a channel in Slack.

**Value:** Cuts Mean Time to Resolution (MTTR) from hours to minutes




# Good Luck
