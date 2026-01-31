# MultiversX AI Skills (mx-ai-skills)

> **Empowering AI Agents with specialized knowledge, roles, and workflows for the MultiversX ecosystem.**

This repository is a central hub for **MultiversX AI Expertise**. It provides a structured collection of specialized "Skills", "Global Workflows" (Roles), and "Documentation" designed to "equip" AI agents (like Antigravity) with the deep technical knowledge required to build, audit, and optimize on MultiversX.

---

## 🚀 Overview
> **AI Agents with specialized knowledge, roles, and workflows for the MultiversX ecosystem.**

This repository is a central hub for **MultiversX AI Expertise**. It provides a structured collection of specialized "Skills", "Global Workflows" (Roles), and "Documentation" designed to equip AI agents with the deep technical knowledge required to build, audit, and optimize on MultiversX.

---

## Quick Install

Install skills for any AI coding agent (Cursor, Windsurf, Codex, etc.):

```bash
npx openskills install michavie/mx-ai-skills
```

Or using Vercel's skills CLI:

```bash
npx skills install michavie/mx-ai-skills
```

---

## Overview

Developing on MultiversX requires specific expertise in Rust (via `multiversx-sc`), WASM optimization, sharding awareness, and unique token standards (ESDT). This repository formalizes that expertise into machine-readable and agent-executable modules.

### Core Value Props:
-   **Role-Based Execution**: Instantiate specialized agents like *MultiversX Smart Contract Developer* or *Full Stack Auditor*.
-   **Granular Skills**: Provide specific toolsets for tasks like *Gas Optimization*, *Static Analysis*, or *Mandos Scenario Testing*.
-   **Security First**: Built-in rules for arithmetic safety, reentrancy protection, and MultiversX-specific "sharp edges".
-   **Documentation Engine**: A curated library of protocol specs and best practices.

---

## 🛠 Repository Structure

The core logic resides in the `skills/` directory, structured for easy ingestion by AI agents:

```text
.
├── skills/
│   ├── contracts/       # Smart Contract Engineer
│   ├── frontend/        # dApp Developer
│   ├── backend/         # Backend Engineer
│   ├── protocol/        # Protocol Architect
│   └── expert/          # 40+ granular expert modules
├── antigravity/
│   ├── global_workflows/    # Specialized Agent Roles (Roleplay instructions)
│   ├── mvx_docs/            # Curated Protocol Documentation
│   └── GEMINI.md            # Global Workspace Rules
└── README.md                # You are here
```

---

## 🧩 Core Components

### 1. Global Workflows (Roles)
Roles defined in `antigravity/global_workflows/` provide the "Persona" and "Protocol" for specific tasks.
-   **`/mvx-smart-contract-developer`**: Focused on idiomatic Rust, gas efficiency, and storage mappers.
-   **`/mvx-auditor`**: A rigorous framework for vulnerability research and system flow verification.
-   **`/mvx-dapp-architect`**: Expert in React integration, `sdk-dapp`, and frontend security.

### 2. Specialized Skills
Skills in `antigravity/skills/` are sets of instructions and checklists for specific activities:
-   **`mvx_static_analysis`**: Patterns for finding unsafe code or floating-point arithmetic.
-   **`consult_mvx_docs`**: High-speed lookup for MultiversX API and framework details.
-   **`mvx_testing_handbook`**: Guidelines for Mandos scenarios and RustVM unit tests.

### 3. Global Rules (`GEMINI.md`)
The `GEMINI.md` file defines the **fundamental behavioral constraints** for AI agents in this workspace, ensuring they:
-   Never hallucinate (Always ask for clarification).
-   Prioritize arithmetic safety (Checked operations only).
-   Follow the Checks-Effects-Interactions pattern strictly.

---

## 📖 How to Use

### For AI Agents (like Antigravity)
These skills are automatically mapped to your workspace. You can invoke specific roles or skills using the `@` symbol or by referencing the workflow name in your tasks.

### For Developers
1.  **Read the Rules**: Familiarize yourself with `GEMINI.md` to understand the standard of code expected.
2.  **Explore Skills**: If you are performing a specific task (e.g., auditing), read the corresponding skill (e.g., `antigravity/skills/mvx_dapp_audit/SKILL.md`).
3.  **Invoke Roles**: Use the slash commands defined in the workflows (e.g., `/mvx-auditor`) to guide the AI's behavior for complex tasks.

---

## 📜 Principles & Standards

-   **Code is Law**: We prioritize security over convenience.
-   **Gas Efficiency**: Every storage write is analyzed.
-   **Reproducibility**: All builds and tests must be traceable and repeatable.
-   **Anti-Hallucination**: If the documentation is missing, the agent MUST ask for data rather than guessing.

---

## 📄 License
- **Universal Compatibility**: Works with 30+ AI coding agents via OpenSkills standard
- **Role-Based Execution**: Instantiate specialized agents like *MultiversX Smart Contract Developer* or *Full Stack Auditor*
- **Granular Skills**: Specific toolsets for *Security Auditing*, *Static Analysis*, *Property Testing*, and more
- **Security First**: Built-in rules for arithmetic safety, reentrancy protection, and MultiversX-specific "sharp edges"

---

## Repository Structure

```text
mx-ai-skills/
├── skills/
│   ├── contracts/              # Rust Smart Contracts Expert
│   ├── frontend/               # React & sdk-dapp Expert
│   ├── backend/                # Go/Py/Node SDKs Expert
│   └── protocol/               # Core Protocol Architecture
│
├── antigravity/                # Agent global workflows
│   ├── global_workflows/       # Agent Roles/Personas
│   ├── mvx_docs/               # Curated documentation
│   └── GEMINI.md               # Global rules
│
├── README.md
└── LICENSE
```

---

## Available Skills

### Primary Skills
| Skill | Description |
|-------|-------------|
| `contracts` | **Smart Contract Engineer**. Build, test, specificy, and audit Rust contracts. Includes security checks and gas optimization. |
| `frontend` | **dApp Developer**. Integrate React/Next.js apps with `sdk-dapp`. Handle login, signing, and state. |
| `backend` | **Backend Engineer**. Build microservices, observers, and scripts using Go, Python, or TypeScript (NestJS). |
| `protocol` | **Protocol Architect**. Deep understanding of Sharding, ESDT mechanics, consensus, and network configuration. |


---

## Usage

### For Any AI Coding Agent (Recommended)

```bash
# Install globally
npx openskills install michavie/mx-ai-skills -g

# Or install in current project
npx openskills install michavie/mx-ai-skills
```


### Manual Installation

```bash
# Clone to your project
git clone https://github.com/michavie/mx-ai-skills.git .ai-skills

# Copy skills to your agent's expected location
cp -r .ai-skills/skills ~/.cursor/skills/        # Cursor
cp -r .ai-skills/skills ~/.agent/skills/         # Generic
```

---

## Principles & Standards

- **Never Hallucinate**: Always ask for clarification when uncertain
- **Security First**: Checked arithmetic, input validation, CEI pattern
- **Gas Efficiency**: Minimize storage writes, optimize WASM size
- **Reproducibility**: All builds and tests must be traceable

---

## Contributing

1. Add new skills to `skills/<skill-name>/SKILL.md`
2. Follow the YAML frontmatter format:
   ```yaml
   ---
   name: skill-name
   description: Brief description. Use when [trigger conditions].
   ---
   ```
3. Use kebab-case for skill names
4. Include practical code examples and checklists

---

## License

This repository is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
