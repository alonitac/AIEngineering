# Agent Skills

Agent Skills are a lightweight, open format for extending AI agent capabilities with specialized knowledge and workflows.

At its core, a skill is a folder containing a `SKILL.md` file that tells the agent how to perform a specific type of task.

Think of it like giving the agent a cheat sheet: every time you ask it to do something that matches a skill's description, it automatically reads that cheat sheet before acting.


## How It Works

When you send a message to the agent, it scans all installed skills and checks if any of their descriptions match what you're asking. **Only if there's a match**, the agent reads the full `SKILL.md` instructions and uses them to guide its response.

You stop repeating the same instructions in every prompt. The agent consistently follows your project's conventions.


## Skill Structure

A skill is just a folder inside `.agents/skills/`:

```text
.agents/skills/
  └── my-skill/
        ├── SKILL.md          # Required: metadata + instructions
        ├── scripts/          # Optional: helper scripts
        ├── references/       # Optional: docs, examples
        └── assets/           # Optional: templates, resources
```

The `SKILL.md` file has two parts - a YAML header and the actual instructions:

```markdown
---
name: my-skill
description: Explain when this skill should activate (1-2 sentences)
---

# My Skill

Instructions for the agent go here.
Write them exactly as you'd write them in a prompt.
```

The `description` is what the agent uses to decide whether to load the skill. Write it so it clearly captures when the skill applies.


## Evaluating Skills

Once you write a skill, how do you know it actually works?  Running structured evaluations (evals) answers these questions and gives you a feedback loop for improving the skill systematically.

### Designing Test Cases

A test case has three parts:

- **Prompt**: a realistic user message - the kind of thing someone would actually type.
- **Expected output**: a human-readable description of what success looks like.
- **Input files** (optional): files the skill needs to work with.

Store test cases in `evals/evals.json` inside your skill directory:

```text
.agents/skills/
  └── yolo-tests/
        ├── SKILL.md
        └── evals/
              ├── evals.json
              └── files/        # optional input files
```

```json
{
  "skill_name": "yolo-tests",
  "evals": [
    {
      "id": 1,
      "prompt": "write a test for the /predict endpoint",
      "expected_output": "A pytest test that mocks the YOLO model, uses a temporary SQLite database, and checks that the response status is 200."
    },
    {
      "id": 2,
      "prompt": "add a test for the /health endpoint",
      "expected_output": "A pytest test, no real model loaded, file named test_health.py"
    }
  ]
}
```

> [!NOTE]
> Currently, there is no standard automated evaluation framework for skills. You can run the tests manually by asking the agent to perform the tasks and checking if the output matches your expectations.



## Community Skills

There's a large open ecosystem of ready-made skills you can install. Browse them at **https://skills.sh/**.

Some popular collections:

| Repo | What's in it |
|------|-------------|
| [supercorp-ai/superpower](https://github.com/supercorp-ai/superpower) | The most-starred collection - web search, browser control, PDFs, email, and more |
| [ComposioHQ/awesome-claude-skills](https://github.com/ComposioHQ/awesome-claude-skills) | Skills built around Claude - GitHub, Notion, Slack integrations |
| [vercel-labs/agent-skills](https://github.com/vercel-labs/agent-skills) | React, Next.js, and Vercel deployment best practices |

To install any skill:

```bash
npx skills add <owner/repo@skill-name>          # project-local (.agents/skills/)
npx skills add <owner/repo@skill-name> -g        # global (~/.agents/skills/)
```

Project-local skills are committed with the repo and shared with your team. Global skills apply to every project on your machine.



# Exercises

### :pencil2: Create a YOLO API testing skill

Create a skill that will help the coding agent to write API tests for the YOLO service. 

1. Create the file `.agents/skills/yolo-api-tests/SKILL.md` in your PolyAI repo. The skill should instruct the agent to:
   - Use `pytest` (or `unittest`) to test the HTTP API.
   - Use a temporary SQLite database, never the real one
   - Assert both the HTTP status code and the response body structure
   - Name test files starting with `test_`
   - Mock the YOLO model
   - Optionally, use `pydantic` models to validate the response body

2. Open the Copilot Chat panel and simply ask:
   ```
   write API tests for the GET /predictions/label/{label} endpoint
   ```
   Check the response - did the agent write HTTP-level tests using your instructions?

3. Temporarily disable the skill:
   ```bash
   mv .agents/skills/yolo-api-tests .agents/skills/_yolo-api-tests
   ```
   Ask the same question again. The agent should now make different choices - probably testing internal functions directly instead of the HTTP API.

4. Re-enable the skill:
   ```bash
   mv .agents/skills/_yolo-api-tests .agents/skills/yolo-api-tests
   ```

> [!TIP]
> The `description` field is the most important part. If it's vague, the skill won't activate when you expect it to. Be specific about the scenarios where the skill applies.

