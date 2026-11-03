# Role Configuration File Description

This directory contains all role configuration files. Each role defines the AI's behavior mode and available tools.

## Creating a New Role

When creating a new role, create a YAML file in the `roles/` directory with the following format:

**Method 1: Explicitly specify the tool list (recommended)**
```yaml
name: Role name
description: Role description
user_prompt: User prompt (appended before the user message to guide AI behavior)
icon: "Icon (optional)"
tools:
    # Add the tools you need...
    # ⚠️ Important: It is recommended to include the following core built-in MCP tools (vulnerabilities and knowledge base)
    - record_vulnerability
    - list_knowledge_risk_types
    - search_knowledge_base
enabled: true
```

**Method 2: Do not set the tools field (use all enabled tools)**
```yaml
name: Role name
description: Role description
user_prompt: User prompt (appended before the user message to guide AI behavior)
icon: "Icon (optional)"
# Do not set the tools field; all tools enabled in MCP management will be used by default
enabled: true
```

## ⚠️ Important Reminder: Core Built-in MCP Tools

**If the `tools` field is set, be sure to include the following tools in the list (at least these three):**

1. **`record_vulnerability`** - Vulnerability management tool, used to record discovered vulnerabilities
2. **`list_knowledge_risk_types`** - Knowledge base tool, lists available risk types
3. **`search_knowledge_base`** - Knowledge base tool, searches knowledge base content

As needed, you can also add other built-in or external tools such as WebShell and batch tasks (subject to those enabled in MCP management).

**Skills (skill packages)**: In **multi-agent / Eino** sessions, the built-in **`skill`** tool loads packages under `skills_dir` on demand; this is not bound to the role YAML.

**Note**: If the `tools` field is not set, the system will use all tools enabled in MCP management by default. To explicitly control the tools available to a role, it is recommended to set the `tools` field explicitly.

## Role Configuration Field Description

- **name**: Role name (required)
- **description**: Role description (required)
- **user_prompt**: User prompt, appended before the user message to guide the AI to adopt specific testing methods and focus areas (optional)
- **icon**: Role icon, supports Unicode emoji (optional)
- **tools**: Tool list, specifies the tools available to this role (optional)
  - **If the `tools` field is not set**: **All tools enabled in MCP management** are selected by default
  - **If the `tools` field is set**: Only the tools specified in the list are used (it is recommended to include at least the core built-in tools mentioned above)
- **enabled**: Whether to enable this role (required, true/false)

## Examples

Refer to the other role files in this directory, such as `Penetration Testing.yaml`, `Web Application Scanning.yaml`, etc.
