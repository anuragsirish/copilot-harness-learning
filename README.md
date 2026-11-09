# Microsoft Copilot Harness in Foundry: Learning Kit

> **Write a skill in Markdown, wire it up in YAML, run `azd up`, and you get a working, governed agent.**
> The **Microsoft Copilot harness** is the managed runtime that Microsoft Foundry runs *around* your prompt agent: the agent loop, a sandbox with a per-conversation workspace, built-in file, shell, web-fetch and subagent tools, and tool orchestration. You don't have to build any of it.

<p align="center">
  <img src="assets/how-it-works.svg" alt="Animated diagram: your app calls the agent endpoint; the Copilot harness runs the agent loop, delegates to subagents, and uses a sandbox, then calls the model, tools and Foundry Files, all under Foundry governance (Entra ID and RBAC, guardrails and egress rules, versions, traces)" width="100%">
</p>

> ⚠️ **Preview.** The Copilot harness (`harness.type: microsoft_copilot_preview`) is in public preview. It has no SLA, and its behavior and APIs may change.

---

## ✅ When to use it

| 👍 Use the Copilot harness for | 👎 Skip it (for now) if |
|---|---|
| **Multi-step tool, skill, file or command workflows** (Learn's wording) | You need **GA and an SLA** today |
| Work that needs **files, shell commands or web-page fetching** in a sandbox | You must bring **your own model** or runtime |
| **Fast, low-code builds** from just Markdown and YAML | You want **full custom code** (orchestration, frameworks, middleware) |
| **Skills** whose scripts and resources need a workspace to run in | Instructions, declared tools and skills alone are enough (a plain prompt agent already handles them) |
| Delegating parts of a task to **subagents** | You need **Foundry Memory** or control over request settings such as `temperature`, `max_tool_calls` or `text.format` (not supported with the harness) |

### Where it sits

```
 Simple ──────────────────────────────────────────────────────────── Full control
 Prompt agent            ⚡ Prompt agent + Copilot harness          Hosted agent
 instructions · tools    + workspace · shell · files ·              your own code & container
 · skills                  web fetch · subagents
```

**Rule of thumb:** start with a prompt agent, which can already call tools several times and use skills. Choose the **Copilot harness** (when you *create* the agent; you can't switch later) once it needs a workspace: read and write files, run shell commands, fetch web pages, or delegate to subagents. Move to a **hosted agent** only when you need code-level control.

---

## 🤔 What problem does it solve?

If you built this kind of agent yourself (for example as a hosted agent), you'd write and run all of this:

| If you build it yourself, you write… | With the harness, Foundry runs… |
|---|---|
| A loop that plans → calls tools → reads results → repeats | 🔁 **Agent loop** (a plain prompt agent's runtime also loops over declared tools; the harness adds built-in tools) |
| A safe place to create files and run shell commands | 📦 **Service-managed, isolated sandbox** |
| Somewhere to keep working state (notes, intermediate files) across turns | 🗂️ **Per-conversation workspace**. Conversation history itself is managed by Foundry for *every* prompt agent, with or without the harness |
| Delegation to helper agents | 🤝 **Subagents** |
| Storage for outputs the agent creates | 📁 **Foundry Files** (agent identity) |
| Auth, RBAC, tracing and evaluation hooks | 🛡️ **Foundry governance**, the same as for any prompt agent (the harness doesn't change it) |

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
| **2** | Define the agent and select the harness | `azure.yaml` (`kind: prompt`, `harness.type: microsoft_copilot_preview`) |
| **3** | Deploy and invoke | `azd up` (or `azd deploy --all`) → `azd ai agent invoke` |
| ↻ | Refine and redeploy | each deploy creates new **immutable versions** |

```bash
azd extension install microsoft.foundry
export AZD_ALPHA_ENABLE_AGENTS_MANAGED_HARNESS=true
azd ai agent init my-agent --no-prompt --kind prompt --agent-name my-agent \
  --project-id "$PROJECT_ID" --model-deployment "$MODEL_DEPLOYMENT"
cd my-agent
# In azure.yaml, add to the agent service:  harness: { type: microsoft_copilot_preview }
azd up --no-prompt
azd ai agent invoke --new-conversation --no-prompt "Give me a research brief on <topic>"
```

> The notebook passes `--harness microsoft_copilot_preview` to `azd ai agent init`. That worked in our run, but Learn documents editing `azure.yaml` instead.

---

## 💡 Example use cases

### From Microsoft (Learn and the official samples)

MS Learn describes the harness as being for *"multi-step tool, skill, file, or command workflows"*. Learn has quickstarts for **web research** and **browser automation** and links to the **CSV Chart Analyst** sample. Microsoft's [official harness samples](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/cli/prompt-agents/microsoft-copilot-harness) show what that looks like:

| | Official sample | What it shows | Why the harness, not a plain prompt agent or a hosted agent |
|---|---|---|---|
| 🔎 | [**Web Research**](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/cli/prompt-agents/microsoft-copilot-harness/quickstart-web-research) (the [Learn quickstart](https://learn.microsoft.com/azure/foundry/agents/quickstarts/create-web-research-agent?pivots=azd)) | Web Search plus a reusable official-sources **skill** | **For a quick cited summary, a plain prompt agent with web search is enough.** The harness pays off for *deep* research: the **Web** tool fetches **full pages**, the agent can save notes to files in its **sandbox workspace**, delegate subtopics to **subagents**, and return the brief **as a file** (these are my suggestions; the quickstart itself only uses web search and a skill) |
| 📧 | [**Web Research Email Digest**](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/cli/prompt-agents/microsoft-copilot-harness/web-research-email-digest) | Web Search, then a **Work IQ Mail** email delivered on a daily schedule | A scheduled job runs **research → write → send** without a person in the loop. With a hosted agent you'd write and host that pipeline yourself |
| 🌐 | [**Browser Automation Blog Research**](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/cli/prompt-agents/microsoft-copilot-harness/browser-automation-blog-research) (the [Learn quickstart](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/browser-automation-copilot-harness-quickstart)) | Managed **browser sessions** (Playwright Workspaces) through a toolbox. The sample turns the harness's built-in tools off so the agent must use the browser tools | Handles sites that need real browsing (clicks, JavaScript-rendered pages) that search snippets can't cover, with no browser fleet to run |
| 🗄️ | [**AdventureWorks SQL Analyst**](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/cli/prompt-agents/microsoft-copilot-harness/adventureworks-sql-analyst) | **Read-only** SQL analysis through managed MCP connector tools, with a skill that loads instructions only as needed | Multi-step querying where the agent explores, queries and refines. Read-only tools and RBAC limit what it can touch |
| 📊 | [**CSV Chart Analyst**](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/cli/prompt-agents/microsoft-copilot-harness/csv-chart-analyst) (linked from Learn) | The uploaded CSV is placed in the workspace, analysed with **shell**, and returned as a PNG through **Foundry Files**, *without Code Interpreter* | Shows the harness's main difference most clearly: a **per-conversation workspace, shell commands and generated files**, saved under the agent's identity |

### Other patterns to consider (my suggestions, not from Microsoft docs)

These fit the harness's capabilities but **aren't documented Microsoft scenarios**. Validate them before using them with customers.

- 📄 **Document processing:** read → transform → write output files to Foundry Files.
- 🧰 **Ops runbooks:** a skill encodes the procedure, MCP tools act, and egress rules limit which hosts it can reach.
- 🤝 **Long multi-step tasks:** the agent loop delegates parts of the work to subagents (Learn: *"Delegate work to additional agents"*).

---

## 🆚 Why the harness? Prompt agent vs. + Copilot harness vs. hosted agent

You *can* build most of these with a plain prompt agent or a hosted agent. The harness sits in the middle: **hosted-agent capabilities with prompt-agent effort**.

| Capability | Prompt agent | **Prompt agent + Copilot harness** | Hosted agent |
|---|---|---|---|
| Authoring | Instructions + tools + skills | **Same, plus built-in harness tools** | Your code and framework |
| Multi-step tool loop | ✅ Multiple model and tool calls with declared tools | ✅ **Managed loop that also uses the built-in tools** | You build it |
| Per-conversation file workspace | ❌ | ✅ **Isolated, persists across turns**. This is a temporary folder in the Foundry sandbox, not your machine or SharePoint, and outputs worth keeping go to Foundry Files | You build it |
| Shell commands and file read/write | Only through declared tools such as Code Interpreter | ✅ **Built-in harness tools** | You build it |
| Subagents | ❌ Not built in | ✅ **Built in** | You build it |
| Return generated files | Through declared tools such as Code Interpreter | ✅ **Uploaded to Foundry Files with the agent's identity** | You build it |
| Network egress control (preview) | ❌ Doesn't apply | ✅ **Allow and deny rules per host** | Your network setup |
| Foundry Memory | Supported | ❌ **Not supported** (conversation history and workspace files are unaffected) | Your choice |
| Runtime to operate | None | **None, because Foundry runs it** | Container, scaling, patching |
| Custom framework, middleware, own model | ❌ | ❌ | ✅ |

### 🛡️ Guardrails the harness gives you

The first three are specific to the harness. The rest are Foundry controls that keep working when you choose the harness.

| Guardrail | How the harness provides it |
|---|---|
| **Isolated execution** | Shell and code run in a **Foundry-managed sandbox**, never on your infrastructure, and each conversation has its own workspace |
| **Least-privilege tools** | Turn each harness tool (**file read, file write, shell, web, subagents**) on or off separately. For example, disable Shell for a read-only agent. **Shell is the broadest tool:** turning off file or Web tools while Shell stays on *isn't* a strict security boundary |
| **Network egress control** | Attach a guardrail policy with allow and deny rules per host to control outbound traffic from shell commands and the Web tool. Start in **Audit** mode, then switch to **Enforce**. Declared tools such as web search and Code Interpreter keep their own network boundaries |
| **Identity, not secrets** | The agent has its own **Entra identity**, so file uploads and toolbox calls are authorized by RBAC (Foundry User role), not keys |
| **Human approval** | Customer MCP tools support `require_approval`. Approval for the harness tools themselves is managed by the service |
| **Versioned and auditable** | Agents and skills are **versioned**, and the usual Foundry project roles, policies and tracing all still apply |

> **In short:** choose a **prompt agent** for simple Q&A, the **Copilot harness** when the agent needs a workspace (files, shell, web fetch, subagents) without you running a runtime, and a **hosted agent** only when you need to own the code, the framework or the model.
>
> <sub>Source: [Use the Microsoft Copilot harness with prompt agents](https://learn.microsoft.com/azure/foundry/agents/how-to/use-tools-skills-copilot-harness), specifically "Understand what the harness changes", "Configure harness tools" and "Control network egress".</sub>

---

## ✅ Your security checklist

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
| [`assets/how-it-works.svg`](assets/how-it-works.svg) | The animated diagram above |
| [`.env.example`](.env.example) | Template for your Foundry project settings |

## ▶️ Run the notebook

**Prerequisites**

- An existing **Foundry project** in a region that supports the Copilot harness, such as **Sweden Central**, **East US** or **Canada Central** ([full list](https://learn.microsoft.com/azure/foundry/agents/concepts/limits-quotas-regions#microsoft-copilot-harness-preview)).
- A model deployed in that project that supports **reasoning and tool calling**.
- **Azure Developer CLI ≥ 1.37** (`brew tap azure/azd && brew install azd` or `winget install microsoft.azd`) with the Foundry extensions (`azd extension install microsoft.foundry`; `azure.ai.agents` ≥ 1.0.0-beta.22).
- The **Foundry User** role on the project for you. To return generated files, also give the agent's identity (`instance_identity.principal_id`) the Foundry User role, and allow up to 5 minutes for it to take effect ([how-to](https://learn.microsoft.com/azure/foundry/agents/how-to/use-tools-skills-copilot-harness)). The notebook doesn't do this step.
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

## 🙏 Credits

The notebook's research skill adapts the core steps of Microsoft's [quickstart-web-research sample](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/cli/prompt-agents/microsoft-copilot-harness/quickstart-web-research) from [microsoft-foundry/foundry-samples](https://github.com/microsoft-foundry/foundry-samples) (MIT License). Everything else in this repo is original.

## 📚 Learn more

- [Use the Microsoft Copilot harness with prompt agents](https://learn.microsoft.com/azure/foundry/agents/how-to/use-tools-skills-copilot-harness)
- [Quickstart: Create a web research agent (azd)](https://learn.microsoft.com/azure/foundry/agents/quickstarts/create-web-research-agent?pivots=azd)
- [Quickstart: Use Browser Automation with the Copilot harness](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/browser-automation-copilot-harness-quickstart)
- [What are prompt agents?](https://learn.microsoft.com/azure/foundry/agents/concepts/prompt-agents)
- [Copilot harness limits, quotas and regions](https://learn.microsoft.com/azure/foundry/agents/concepts/limits-quotas-regions#microsoft-copilot-harness-preview)
- [Foundry blog: Microsoft Copilot harness in Foundry](https://devblogs.microsoft.com/foundry/microsoft-copilot-harness-in-foundry/)

---

<sub>Created by Anurag Karuparti · CSA, Principal AI Apps Architect</sub>
