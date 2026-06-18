# Intro to AI Coding with GitHub Copilot

AI coding agents have become standard tools in professional software development. Once you use it, there is no going back. 


## What Is an AI Coding Agent?

An AI **coding agent** is not just sending a simple prompt to an LLM and getting a response. It's a software system that runs something called an **agentic loop**:


Goal -> Think -> Act -> Observe -> Think -> Act -> ... -> Done


You give it a task. It reads files, runs commands, writes code, checks results - all on its own, until the task is done or it asks you for guidance.

Three parts make this work:

- **Model** - The LLM doing the reasoning (GPT-4o, Claude, Gemini, etc.) 
- **Harness** - The software infrastructure that wraps around an AI LLM (LangChain, LangGraph, AutoGen, etc.)
- **Context Window** - Everything the model can see right now - files, chat history, tool results 

**AI Agent = Model + Harness** 

The harness decides what goes into the context window. The model decides what to do next. 


## Setup

You need a GitHub account with Copilot enabled.

- **Free tier** - limited usage, enough to get started
- **Individual plan** - paid, full access

Go to **github.com → Settings → Copilot** to activate it.

Open VS Code, press `Ctrl+Shift+X`, search and install **GitHub Copilot Chat** if it isn't installed yet.


## Copilot inline completions

You've probably seen Copilot's inline suggestions before. This is the simplest way to use it: start typing, and it offers code completions.

No agent loop, no tool calls - just a snapshot of your file sent to the model, and a completion back. 
> [!TIP]
> If completions are distracting, turn them off for a specific file type: click the Copilot icon in the status bar and disable/snooze.


## Chat modes


Open the Copilot Chat panel: click the chat icon in the sidebar or press `Ctrl+Alt+I`. 

You can choose different modes for different tasks:

- **Ask** - for understanding code, getting explanations, or brainstorming ideas
- **Agent** - for tasks that require reading multiple files, running commands, and writing code
- **Plan** - when you want the agent to produce a plan first before executing


> [!IMPORTANT] 
> Never commit code you haven't read. You are responsible for everything in the commit.


## Context Window Management

The context window is the agent's "working memory". It contains everything the agent can see at any moment: the current file, other files you attach, the chat history, and tool results.

The model can read files during the session, but it doesn't automatically see everything in your repo. You have to explicitly attach files or directories to the context.

The agent can only work with what it sees. If you ask it to write a test for a function, but it can't see the function's code, it will **hallucinate** something that looks right but is actually wrong, or something that doesn't even exist in your repo. 

This is the whole bible in building AI agents - get the right model, give the model the right context at the right time for the given task.

To do to, we'll learn throughout the course an important concept: **steering, skills, tools, memory, context management, system prompt, subagents, human-in-the-loop.** 

Let's talk about 2 of them now: **steering** and **skills**.

### Steering

By default the agent knows nothing about your project's conventions. It will make reasonable guesses - and those guesses won't always be right. For example, it might write tests using `pytest` when your project uses `unittest`.

The fix is simple: create an `AGENTS.md` file at the root of your repository. Most coding agents load it automatically at the start of every session (Copilot, Codex, Claude, etc.). It's a plain markdown file where you write the rules you'd otherwise repeat in every prompt:

- Clearly outline the core framework, database setup, and architectural style.
- The Git workflow used in the SDLC (feature branches, PRs, etc.)
- Do's and don'ts for code style, testing, and deployment

Take a look at the `AGENTS.md` file in your **PolyAI** repository for a real example.

> [!NOTE]
> Different agents look for different filenames. Copilot uses `.github/copilot-instructions.md`, OpenAI Codex uses `AGENTS.md`, Claude Code uses `CLAUDE.md`. The idea is the same - the filename is just an agent-specific convention. See the [GitHub Copilot docs](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/add-custom-instructions/add-repository-instructions#creating-custom-instructions) for Copilot's full options.

### Skills

Skills are reusable, packaged sets of instructions that tell the agent how to handle a specific type of task. Instead of repeating the same guidance in every prompt, you define it once as a skill and the agent loads it when needed.

We will cover skills in depth in the next tutorial.


## Copilot CLI

Copilot CLI is a command line interface for Copilot. It allows you to run Copilot commands directly from your terminal, without opening VS Code.

Installing with npm (all platforms)

```bash
npm install -g @github/copilot
```

On first launch, if you're not currently logged in to GitHub, you'll be prompted to use the `/login` slash command. Enter this command and follow the on-screen instructions to authenticate. 

1. In your terminal, navigate to a folder that contains code you want to work with.
2. Enter `copilot` to start Copilot CLI.
3. Enter a prompt in the CLI.

When Copilot wants to use a **tool** that could modify or execute files, for example, `touch`, `chmod`, or `sed` - it will ask you to approve the use of the tool.



 