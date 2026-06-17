# Intro to AI Coding with GitHub Copilot

AI coding agents have become standard tools in professional software development. Once you use it, there is no going back. 


## What Is an AI Coding Agent?

An AI **coding agent** is not just sending a simple prompt to an LLM and getting a response. It's a software system that runs a loop:

```
[Goal] → [Think] → [Act] → [Observe] → [Think] → [Act] → ... → [Done]
```

You give it a task. It reads files, runs commands, writes code, checks results - all on its own, until the task is done or it asks you for guidance.

Three parts make this work:

| Part | What it is |
|------|-----------|
| **Model** | The LLM doing the reasoning (GPT-4o, Claude, Gemini, etc.) |
| **Harness** | The software infrastructure that wraps around an AI LLM |
| **Context Window** | Everything the model can see right now - files, chat history, tool results |

The harness decides what goes into the context window. The model decides what to do next. 


## Setup

### 1. GitHub Copilot Subscription

You need a GitHub account with Copilot enabled.

- **Free tier** - limited usage, enough to get started
- **Individual plan** - paid, full access

Go to **github.com → Settings → Copilot** to activate it.

### 2. VS Code Extension

Open VS Code, press `Ctrl+Shift+X`, search and install **GitHub Copilot Chat** if it isn't installed yet.

### 3. Copilot CLI

For working in the terminal, install the GitHub CLI and the Copilot extension:

```bash
# Ubuntu/Debian
sudo apt install gh
```

```bash
gh auth login
gh extension install github/gh-copilot
gh copilot --version
```


## Inline Completions

You've probably seen Copilot's inline suggestions before. This is the simplest way to use it: start typing, and it offers code completions.

No agent loop, no tool calls - just a snapshot of your file sent to the model, and a completion back. 
> [!TIP]
> If completions are distracting, turn them off for a specific file type: click the Copilot icon in the status bar and disable/snooze.


## Chat Modes


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

The agent can only work with what it sees. If you ask it to write a test for a function, but it can't see the function's code, it will **hallucinate** something that looks right but is actually wrong.

### Agent steering

By default the agent knows nothing about your project's conventions. It will make reasonable guesses - and those guesses won't always be right. For example, it might write tests using `pytest` when your project uses `unittest`.

The fix is simple: create an `AGENTS.md` file at the root of your repository. Most coding agents load it automatically at the start of every session (Copilot, Codex, Claude, etc.). It's a plain markdown file where you write the rules you'd otherwise repeat in every prompt:

- Which Python version to use 
- Which test framework to use
- What the git workflow is
- Very high level architecture of the project
- What to never do

Take a look at the `AGENTS.md` file in your **PolyAI** repository for a real example.

> [!NOTE]
> Different agents look for different filenames. Copilot uses `.github/copilot-instructions.md`, OpenAI Codex uses `AGENTS.md`, Claude Code uses `CLAUDE.md`. The idea is the same - the filename is just an agent-specific convention. See the [GitHub Copilot docs](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/add-custom-instructions/add-repository-instructions#creating-custom-instructions) for Copilot's full options.


## Copilot CLI


When you're already in the terminal, the CLI is faster than switching to VS Code.

### `gh copilot suggest`

Get a shell command for something you don't know off the top of your head:

```bash
gh copilot suggest "run only tests in services/yolo/tests/test_predict.py"
```

```bash
gh copilot suggest "find all Python files modified in the last 7 days"
```

### `gh copilot explain`

Understand a command you're not sure about:

```bash
gh copilot explain "find . -name '*.pyc' -delete"
```


 