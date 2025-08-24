### 🔄 Project Awareness & Context
- **Always read `PLANNING.md`** at the start of a new conversation to understand the project's architecture, goals, style, and constraints.
- **Check `TASK.md`** before starting a new task. If the task isn’t listed, add it with a brief description and today's date.
- **Use consistent naming conventions, file structure, and architecture patterns** as described in `PLANNING.md`.
- **Use npm/yarn** for package management and script execution in the TypeScript/Next.js environment.

### 🧱 Code Structure & Modularity
- **Never create a file longer than 500 lines of code.** If a file approaches this limit, refactor by splitting it into modules or helper files.
- **Organize code into clearly separated modules**, grouped by feature or responsibility.
  For agents this looks like:
    - `agent.ts` - Main agent definition and execution logic 
    - `tools.ts` - Tool functions used by the agent 
    - `prompts.ts` - System prompts
    - `types.ts` - TypeScript type definitions
- **Use clear, consistent imports** (prefer relative imports within packages).
- **Use clear, consistent imports** (prefer relative imports within packages).
- **Use Next.js environment variables** (.env.local) and process.env for configuration.

### 🧪 Testing & Reliability
- **Always create Jest unit tests for new features** (functions, components, API routes, etc).
- **After updating any logic**, check whether existing unit tests need to be updated. If so, do it.
- **Tests should live in a `/tests` folder** mirroring the main app structure.
  - Include at least:
    - 1 test for expected use
    - 1 edge case
    - 1 failure case

### ✅ Task Completion
- **Mark completed tasks in `TASK.md`** immediately after finishing them.
- Add new sub-tasks or TODOs discovered during development to `TASK.md` under a “Discovered During Work” section.

### 📎 Style & Conventions
- **Use TypeScript** as the primary language.
- **Follow strict TypeScript**, use type annotations, and format with `Prettier`.
- **Use `Zod` for data validation**.
- Use `Next.js API Routes` for APIs and `Firestore` for database operations if applicable.
- Write **JSDoc for every function** using the TypeScript style:
  ```typescript
  /**
   * Brief summary.
   * 
   * @param param1 - Description
   * @returns Description
   */
  function example(param1: string): ReturnType {
    // implementation
  }
  ```

### 📚 Documentation & Explainability
- **Update `README.md`** when new features are added, dependencies change, or setup steps are modified.
- **Comment non-obvious code** and ensure everything is understandable to a mid-level developer.
- When writing complex logic, **add an inline `# Reason:` comment** explaining the why, not just the what.

### 🧠 AI Behavior Rules
- **Never assume missing context. Ask questions if uncertain.**
- **Never hallucinate libraries or functions** – only use known, verified npm packages from the Next.js/React ecosystem.
- **Always confirm file paths and module names** exist before referencing them in code or tests.
- **Never delete or overwrite existing code** unless explicitly instructed to or if part of a task from `TASK.md`.

### 📖 Learning & Knowledge Management
- **Learn from mistakes and challenges** encountered during development.
- **When finding a working solution** to a problem (bug fix, implementation pattern, configuration, etc.), add it to the `examples/` folder with:
  - A descriptive filename indicating the problem solved
  - Clear documentation of the issue and solution
  - Code examples or configuration snippets
  - Any relevant context or gotchas discovered
- **Reference existing examples** in the `examples/` folder when encountering similar problems.

### 🪟 Windows Development Notes
- **GitHub CLI on Windows**: Use `gh` command directly in PowerShell or Command Prompt, not through bash/WSL
- **Path Issues**: If `gh` command not found, restart terminal after installation or add GitHub CLI to PATH
- **Alternative PR Creation**: If GitHub CLI unavailable, create PRs manually through GitHub web interface