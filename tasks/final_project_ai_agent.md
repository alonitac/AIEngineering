# Final Project

## Overview

In this project you'll design and build an **AI agent** that solves a real-world problem.

The project brings together everything you've learned in the course: agents, MCP, containers, Kubernetes, Terraform, AWS cloud services, CI/CD, and observability.

## Requirements

### 0. Project Planning

Before writing a single line of code, plan your project using the [**Superpowers brainstorming skill**](https://github.com/obra/superpowers) (or any other good equivalent). The process will walk you through clarifying the problem, exploring design alternatives, and producing a written spec and an implementation plan.

Your project repository must contain two documents under `docs/`:

- `docs/spec.md` - the design specification produced by the brainstorming session: problem statement, architecture, components, data flow, error handling, and testing strategy
- `docs/plan.md` - the step-by-step implementation plan produced by the writing-plans skill

Both documents must be committed and approved by course staff (as usual, via a PR) **before you start coding**. They are part of your grade.

### 1. Agent Design

Build an intelligent agent that:

- Solves a **clearly defined, real business problem** with measurable value
- Uses an LLM framework of your choice - LangGraph, LangChain, or equivalent. No no-code platforms (n8n, CrewAI, etc.) - you write the agent code
- Exposes an **HTTP API** at minimum. A web UI is highly recommended
- Well-crafted: retries, graceful termination, fallbacks, clear error responses to the user
- Has a system prompt that defines the agent's persona, capabilities, and boundaries

### 2. MCP Tooling

Your agent must use the **MCP** for tool integration:

- Connect to at least one publicly or self-hosted MCP server. Examples: GitHub MCP, Brave Search MCP, Slack MCP, Filesystem MCP, Kubernetes MCP, Gmail MCP
- You are highly encouraged to build your own MCP server that exposes tools specific to your domain. 

Your agent must call tools during a real interaction.

### 3. Infrastructure & Deployment

**Kubernetes on AWS EC2 - no EKS.** Provision and manage your own cluster on EC2 instances.

- Deploy your full stack in both `dev` and `prod` namespaces with separate configuration per environment
- Well-crafted workloads: liveness/readiness probes, resource requests/limits, HPA (autoscaling), secrets management, ConfigMaps
- Use **AWS cloud services** where they fit your domain: S3 for object storage, DynamoDB for key-value data, SQS for async messaging, SNS for notifications, and so on.
- All stack should follow Infrastructure as Code, i.e., provision all AWS resources using **Terraform**. No manual clicking anywhere!


### 4. CI/CD Pipeline

Configure a full CI/CD pipeline that:

- Runs all tests on every **pull request**
- Deploys to `dev` and `prod`
- Reports test results clearly (Allure, GitHub Actions summary, Codecov, or your tool of choice)

### 5. Observability

Your system must be monitored end-to-end. Define what "healthy" means for your agent, instrument it, and make sure you know when something is wrong before your users do:

- Collect **metrics and logs** from all services
- Set up **alerts** for meaningful conditions: error rate spikes, high latency, LLM failures, tool timeouts
- Provide a **dashboard** that gives a clear picture of system health at a glance

You are free to use any observability stack (Prometheus + Grafana, CloudWatch, Datadog free tier, OpenTelemetry + any backend, etc.).

## Testing

Write automated tests for your project. Provide a clear **test plan** document (what you test, how, and your success criteria). Your tests must include:

- **Unit tests** - test agent logic and MCP server tools in isolation. Mock the LLM and external services
- **Integration tests** - test the interaction between your agent and your local MCP server using the real MCP transport

## Extra

- **Web UI** - a chat interface, dashboard, or any UI that makes your agent accessible to non-technical users
- **Multi-agent** - decompose your system into specialized sub-agents that collaborate to solve complex tasks
- **Agent Skills** - write reusable [Superpowers skills](https://github.com/obra/superpowers) that encode domain knowledge or workflows specific to your project (e.g., a skill for deploying a new version, a skill for triaging alerts). Skills that are genuinely useful to the class will receive extra credit

## Presentation

- Prepare a **15-minute** presentation with **slides** and a **live demo**
- Your slides must include:
  - A short introduction of yourself
  - Your problem statement and why it has real business value
  - Your architecture: agent, MCP servers, AWS infrastructure, observability
  - Testing overview: what you tested, your strategy, success criteria
  - A dedicated slide reflecting on your experience working with AI coding agents:
    - How did you use AI during this project?
    - What were the biggest challenges when working with an AI agent?
    - What did you understand or decide that the AI couldn't?
    - Which skills did you use, and did they help?
- Show a **live demo** of your agent handling a real request end-to-end
- Show your **observability dashboard** live, with real metrics from your demo
- Show your **CI pipeline** running in GitHub Actions
- Be ready to explain every decision you made - architecture, tools, testing, deployment
- **Bonus:** present in English


## Inspiring Use Cases

The following ideas are meant to inspire you. You are free to implement any of them or invent your own.

[Awesome MCP servers](https://github.com/punkpeye/awesome-mcp-servers) or [MCP Servers](https://mcpservers.org/)

### DevOps Incident Response Agent

**Problem:** On-call engineers spend too much time manually correlating logs, metrics, and alerts to diagnose production incidents - often at 3 AM.

**What the agent does:** Receives a Prometheus alert, fetches the relevant pod logs, queries recent metrics, identifies the likely root cause, and suggests a fix as a PR, or opens a GitHub issue or pages a channel in Slack.

**Value:** Cuts Mean Time to Resolution (MTTR) from hours to minutes

### Interactive Docs Agent

**Problem:** Documentation sites are static — you read, guess, copy-paste, get an error, and repeat. There is no one to ask "why doesn't this work *in my project*?"

**What the agent does:** Pick a docs site — FastAPI, FastMCP, Kubernetes, LangGraph, Terraform, or any other. The agent indexes the docs and answers questions with awareness of the developer's actual GitHub repo. *"Why is my MCP tool not being called?"* → the agent reads the student's MCP server code, cross-references it with the FastMCP docs, and pinpoints the bug. *"How do I add auth to my FastAPI app?"* → it reads the existing `main.py` and returns a snippet that fits the project's structure, not a generic copy from the docs.


**Business value:** Closes the gap between "I read the docs" and "it works in my project"

### Autonomous Business Operator (Odoo)

**Problem:** Running a business means constantly juggling sales, inventory, purchasing, and finance — each department reacting to the others too slowly, with too much manual coordination in between.

**Base system:** [**Odoo Community Edition**](https://github.com/odoo/odoo) — a free, open-source business platform with CRM, Sales, Inventory, Purchasing, Accounting, HR, and more, all backed by a single documented [JSON-RPC API](https://www.odoo.com/documentation/17.0/developer/reference/external_api.html). Run it locally with `docker-compose up` and load the built-in demo data to get a fully populated business with customers, suppliers, open orders, stock, and invoices — no fixtures needed.

**What the agent does:** The agent acts as a cross-departmental operator that watches the whole business and connects the dots humans miss. A few examples of what it can do autonomously:

- A sales order comes in → agent checks stock → if inventory is low, creates a draft purchase order and pings the procurement manager
- An invoice goes 14 days overdue → agent looks up the customer's communication history in the CRM → drafts a tailored follow-up email → escalates to a human if there's still no response after two attempts
- Ask it *"What are my biggest business risks right now?"* → it queries sales pipeline, stock levels, overdue invoices, and open POs, then returns a prioritized risk briefing

 
**Business value:** One agent that sees the whole business and acts — the kind of coordination that normally takes three departments and a week


# Good Luck
