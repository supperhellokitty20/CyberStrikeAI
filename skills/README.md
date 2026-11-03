# Skills directory (Agent Skills / Eino)

- Each skill is a **subdirectory** containing a **`SKILL.md`** at its root (YAML front matter: `name`, `description`, plus the Markdown body). See [agentskills.io](https://agentskills.io/specification.md).
- The **directory name must match `name`**.
- **Runtime loading**: in an **Eino DeepAgent (multi-agent)** session, the ADK **`skill`** middleware discloses skills progressively (the system prompt lists each skill's name/description; the model then calls the **`skill`** tool to fetch the full `SKILL.md`). Optionally enable **`multi_agent.eino_skills.filesystem_tools`** to use the same `read_file` / `execute` tools as the host for scripts and resources inside a package.
- **Web management**: HTTP `/api/skills/*` remains available for listing, editing, and uploading files inside a package (implemented by `internal/skillpackage`, not MCP).
- **Runtime scope**: skills load progressively through the ADK **`skill`** tool inside multi-agent (DeepAgent) sessions; the single-agent MCP loop does not include Skills. Enable multi-agent mode (or the future single-agent Eino path) to use them.
