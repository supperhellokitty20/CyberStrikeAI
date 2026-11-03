# Citations and External Links (Example)

This file verifies that the **`references/`** directory inside a skill package is recognized by the listing API, HTTP `resource_path`, and the multi-agent host file tools.

## Test Procedure (Inside an Authorized Environment)

1. The `package_files` field in the `GET /api/skills/cyberstrike-eino-demo` response should include `references/citations.md`.
2. `GET /api/skills/cyberstrike-eino-demo?resource_path=references/citations.md` should return the contents of this file.
3. In a multi-agent session with `eino_skills.filesystem_tools` enabled, this file can be read via its relative path.

## Placeholder Citation

- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/) (link-format example only)
