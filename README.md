# Microsoft Copilot Harness in Foundry: Learning Kit

> **Write a skill in Markdown, wire it up in YAML, run `azd deploy`, and you get a production-shaped agent.**
> The **Microsoft Copilot harness** is the managed runtime that Microsoft Foundry runs *around* your prompt agent: the agent loop, a sandbox, context, files and tool orchestration. You don't have to build any of it.

<p align="center">
  <img src="assets/how-it-works.svg" alt="Animated diagram: your app calls the agent endpoint; the Copilot harness runs the agent loop and a sandbox, then calls the model, tools and Foundry Files" width="100%">
</p>

> ⚠️ **Preview.** The Copilot harness (`harness.type: microsoft_copilot_preview`) is in public preview. It has no SLA, and its behavior and APIs may change.

---

## ✅ When to use it

| 👍 Use the Copilot harness for | 👎 Skip it (for now) if |
|---|---|
| **Multi-step, tool-heavy tasks** like research, analysis or triage | You need **GA and an SLA** today |
| Work that needs **files and code execution** in a sandbox | You must bring **your own model** or runtime |
| **Fast, low-code builds** from just Markdown and YAML | You want **full custom code** (orchestration, frameworks) |
| Reusable know-how packaged as **skills** | A single **simple Q&A** prompt is all you need |

### Where it sits

```
 Simple ──────────────────────────────────────────────────────────── Full control
 Prompt agent            ⚡ Prompt agent + Copilot harness          Hosted agent
 Q&A, light tools        tools · files · sandbox · skills           your own code & container
```

**Rule of thumb:** start with a prompt agent. Add the **Copilot harness** once the agent needs to *do work*: run several steps, call tools, read and write files, or run code. Move to a **hosted agent** only when you need code-level control.

---

## 🤔 What problem does it solve?

Building a real agent usually means writing and running all of this yourself:

| Without the harness, you build… | With the harness, Microsoft runs… |
|---|---|
| A loop that plans → calls tools → reads results → repeats | 🔁 **Agent loop** |
| A safe place to create files and run shell or code | 📦 **Isolated sandbox** for each conversation |
| Conversation memory and context-window management | 🧠 **Context management** |
| Delegation to helper agents | 🤝 **Subagents** |
| Storage for outputs the agent creates | 📁 **Foundry Files** (agent identity) |
| Auth, RBAC, tracing and evaluation hooks | 🛡️ **Foundry governance** |

**You bring only three things:**

1. 📝 **Instructions**: who the agent is and how it should behave.
2. 🧩 **Skills**: reusable procedures written as `SKILL.md` files.
3. 🛠️ **Tools**: web search, MCP servers and functions.

You also pick a model that supports **reasoning and tool calling**, such as `gpt-6.1-terra`.

---

## 🚀 How it works in three steps

| Step | What you do | Artifact |
|---|---|---|
| **1** | Write the skill | `skills/<skill>/SKILL.md` |
| **2** | Define the agent and select the harness | `azure.yaml` (`kind: prompt`, `harness: microsoft_copilot_preview`) |
| **3** | Deploy and invoke | `azd deploy --all` → `azd ai agent invoke` |
| ↻ | Refine and redeploy | each deploy creates new **immutable versions** |

```bash
export AZD_ALPHA_ENABLE_AGENTS_MANAGED_HARNESS=true
azd ai agent init my-agent --harness microsoft_copilot_preview
azd deploy --all
azd ai agent invoke --new-conversation "Give me a research brief on <topic>"
```

---

## 💡 Example use cases

| | Use case | What the harness adds |
|---|---|---|
| 🔎 | **Research briefs** | Web search, then a structured, cited write-up |
| 📊 | **Data analysis** | Reads uploaded CSVs and runs code in the sandbox to build charts and stats |
| 📄 | **Document processing** | Extracts, transforms and saves outputs to Foundry Files |
| 🧰 | **Ops runbooks** | Follows a skill step by step and calls tools through MCP |
| 🎧 | **Support agents** | Combines knowledge, tools and per-conversation context |
| 🤝 | **Multi-step work** | Plans, delegates to subagents and checks its own results |

---

## 🛡️ Guardrails to apply from day one

- 🐚 **Limit the shell.** Enable only the tools the agent really needs.
- 🪪 **Identity, not secrets.** Use the agent's Entra identity and RBAC, never keys in instructions.
- 🌐 **Control egress.** Restrict where the agent can reach.
- 🧪 **Treat content as data.** Web pages and files are untrusted input, not instructions.
- ✅ **Approve MCP calls.** Require approval for sensitive tools.
- 🔍 **Trace everything.** Use Foundry tracing and evaluations.

---

## 📦 What's in this repo

| File | Purpose |
|---|---|
| [`copilot_harness_learning.ipynb`](copilot_harness_learning.ipynb) | **Runnable notebook** following the official Learn flow: SKILL.md → azure.yaml → `azd deploy` → invoke (azd and SDK) → refine and redeploy → optional cleanup |
| [`copilot_harness_slides.html`](copilot_harness_slides.html) | Animated 9-slide deck explaining why the harness matters and when to use it. Open it in a browser and use ← / → to move between slides |
| [`build_notebook.py`](build_notebook.py) | Generates the notebook. Edit this file, then re-run it |
| [`assets/how-it-works.svg`](assets/how-it-works.svg) | The animated diagram above |
| [`.env.example`](.env.example) | Template for your Foundry project settings |

## ▶️ Run the notebook

**Prerequisites**

- An existing **Foundry project** in a region that supports the Copilot harness, such as **Sweden Central**, **East US** or **Canada Central**.
- A model deployed in that project that supports **reasoning and tool calling**.
- **Azure Developer CLI ≥ 1.37** (`brew install --cask azure/azd/azd` or `winget install microsoft.azd`) with the Foundry extensions.
- The Azure CLI, signed in with `az login` and `azd auth login`.
- Python 3.10+.

**Steps**

```bash
git clone https://github.com/anuragsirish/copilot-harness-learning.git
cd copilot-harness-learning
python -m venv .venv && source .venv/bin/activate
pip install jupyter nbformat pyyaml "azure-ai-projects>=2.10" azure-identity openai
cp .env.example .env      # fill in your subscription, resource group, Foundry resource and project
code copilot_harness_learning.ipynb   # or: jupyter lab
```

> 💡 **Tip:** To call a harness agent from the SDK, use `AIProjectClient(...).get_openai_client(agent_name=...)`. That is the **agent-scoped** endpoint. The project-level endpoint returns HTTP 400 for harness agents.

> 💸 Running the notebook **deploys a real agent and skill** to your project. Set `CLEANUP = True` in the last section to delete them afterwards.

## 📚 Learn more

- [Use the Microsoft Copilot harness with prompt agents](https://learn.microsoft.com/azure/foundry/agents/how-to/use-tools-skills-copilot-harness)
- [Quickstart: Create a web research agent (azd)](https://learn.microsoft.com/azure/foundry/agents/quickstarts/create-web-research-agent?pivots=azd)
- [Copilot harness limits, quotas and regions](https://learn.microsoft.com/azure/foundry/agents/concepts/limits-quotas-regions#microsoft-copilot-harness-preview)
- [Foundry blog: Microsoft Copilot harness in Foundry](https://devblogs.microsoft.com/foundry/microsoft-copilot-harness-in-foundry/)

---

<sub>Created by Anurag Karuparti · CSA, Principal AI Apps Architect</sub>
