# Generative AI & Agents
## File 06: GitHub Copilot Basics & Best Practices

---

## What You'll Learn
- How GitHub Copilot Agentic Mode (GA, April 2026) works inside your IDE.
- The new autonomous Copilot Coding Agent — Issue-to-PR workflow.
- Best practices for prompting Copilot for code.
- Slash commands, context variables, and `.agent.md` customization.

**Time Required:** 30 minutes

---

## 1. How Copilot Works Under the Hood

GitHub Copilot is not just sending your prompt to OpenAI. It is deeply integrated into your IDE (VS Code, Visual Studio, etc.) and performs **Prompt Injection** before the LLM ever sees your request.

When you ask Copilot a question, it secretly attaches:
1. The file you are currently looking at.
2. Other open tabs in your IDE.
3. Information about your project's framework (e.g., "This is a .NET 8 project").
4. Recent terminal errors.

*Key Takeaway:* **Context is everything.** If you want Copilot to write a controller that uses a specific Repository class, **open the Repository class in another tab** before asking Copilot to write the controller.

---

## 2. The Three Modes of Copilot

1. **Ghost Text (Autocomplete):** As you type, Copilot suggests the next lines of code in gray text. Press `Tab` to accept.
   - *Pro-tip:* Write a descriptive comment first (e.g., `// Connect to Azure SQL using Managed Identity and retry logic`). The ghost text will use the comment as instructions.
   
2. **Inline Chat (`Ctrl+I` / `Cmd+I`):** Highlight a block of code and press `Ctrl+I`. This opens a small chat box floating over your code.
   - Ideal for refactoring: *"Extract this block into a separate private method"* or *"Add try/catch logging here"*.

3. **Copilot Chat Window:** The dedicated sidebar chat. Use this for architectural questions or generating entire new classes.

---

## 3. Slash Commands

Slash commands are shortcuts that tell Copilot exactly what task you want it to perform, optimizing its internal system prompt for that specific job.

- `/explain` — Explains the highlighted code block in plain English.
- `/fix` — Analyzes a highlighted error or bug and proposes a fix.
- `/tests` — Generates unit tests for the highlighted code.
- `/doc` — Generates XML/Docstring comments for the highlighted method.

**Example Usage:**
Highlight your complex LINQ query, open Inline Chat (`Ctrl+I`), and type: `/explain How does the GroupBy work here?`

---

## 4. Context Variables (`@`)

You can explicitly force Copilot to read certain files or workspaces by using the `@` symbol in the Chat window.

- `@workspace` — Tells Copilot to search across your entire repository, not just the open files.
  - *Example:* `@workspace Where do we configure the database connection string?`
- `@vscode` — Asks questions about the IDE itself.
- `@terminal` — Tells Copilot to look at the last error output in your terminal.
- `@github` — **(New, 2026)** Searches across all your GitHub repositories, issues, and PRs.
  - *Example:* `@github find recent PRs that changed the authentication middleware`

---

## 5. Agentic Mode & the Copilot Coding Agent (April 2026 GA)

**Agentic Mode** is now Generally Available in VS Code, Visual Studio, and JetBrains IDEs. It is the biggest shift in how Copilot works.

### What Agentic Mode Can Do
- **Multi-file editing:** Copilot autonomously edits multiple related files in one request.
- **Terminal command execution:** Copilot can run `dotnet build`, `git status`, and test runners.
- **Self-healing:** If a test fails after a code change, Copilot reads the error and retries.
- **Full Issue-to-PR workflow:** Assign a GitHub Issue to the Copilot agent. It will write the code, run tests, and open a Pull Request — completely autonomously.

### Using Agentic Mode in VS Code
1. Open Copilot Chat (`Ctrl+Alt+I`).
2. Switch the dropdown from **Ask** to **Agent**.
3. Type a high-level task: *"Add input validation to all POST endpoints in the Orders controller using `FluentValidation`."*
4. Review the proposed file changes in the diff view. Approve or edit them.

### `.agent.md` — Custom Agent Personas
You can define a custom agent persona directly in your repository using a `.github/agents/` folder:

```markdown
<!-- .github/agents/contoso-api-reviewer.agent.md -->
---
name: Contoso API Reviewer
description: Reviews .NET API code against Contoso's internal standards
---

You are an expert .NET 8 API code reviewer for Contoso Engineering.

ALWAYS check for:
1. FluentValidation on all DTO inputs
2. Serilog structured logging on all exceptions
3. EF Core async methods (never synchronous DB calls)
4. Azure Managed Identity for all external connections

Output findings as a Markdown table with columns: File | Line | Severity | Finding.
```

Now any developer can invoke this with `@contoso-api-reviewer` in the chat.

---

## 🧪 Exercise: The Comment-Driven Development Test

1. Open a blank C# or Python file.
2. Write a descriptive comment at the top: `// A class representing a Shopping Cart that can add items, remove items, and calculate a 10% tax.`
3. Press `Enter` and wait 2 seconds.
4. Keep pressing `Tab` and `Enter` to see how much of the class Copilot can write based purely on that single comment.

---

**Next:** [07-COPILOT-CUSTOM-AGENTS.md](./07-COPILOT-CUSTOM-AGENTS.md) — How to build your own Custom Copilot Agents to automate your company's specific workflows!
