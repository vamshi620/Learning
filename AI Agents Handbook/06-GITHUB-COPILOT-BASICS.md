# Generative AI & Agents
## File 06: GitHub Copilot Basics & Best Practices

---

## What You'll Learn
- How GitHub Copilot works inside your IDE.
- Best practices for prompting Copilot for code.
- Slash commands and context targeting.

**Time Required:** 20 minutes

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
  - *Example:* `@vscode How do I change my theme to dark mode?`
- `@terminal` — Tells Copilot to look at the last error output in your terminal.
  - *Example:* `@terminal Why did my docker build fail?`

---

## 🧪 Exercise: The Comment-Driven Development Test

1. Open a blank C# or Python file.
2. Write a descriptive comment at the top: `// A class representing a Shopping Cart that can add items, remove items, and calculate a 10% tax.`
3. Press `Enter` and wait 2 seconds.
4. Keep pressing `Tab` and `Enter` to see how much of the class Copilot can write based purely on that single comment.

---

**Next:** [07-COPILOT-CUSTOM-AGENTS.md](./07-COPILOT-CUSTOM-AGENTS.md) — How to build your own Custom Copilot Agents to automate your company's specific workflows!
