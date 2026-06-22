# Contributing to ORAXIA

Thank you for your interest in contributing to ORAXIA! We welcome contributions of all kinds.

## How to Contribute

### 1. Reporting Issues
- Search existing issues before opening a new one
- Include Oracle version, APEX version, and AI platform details
- Provide a minimal reproducible example

### 2. Improving Skill Files

Skill files live in `skills/*/SKILL.md`. A great skill contribution:
- Follows Oracle SQL / PL/SQL best practices
- Includes working code examples
- Covers edge cases and anti-patterns
- Includes performance considerations

### 3. Adding a New Skill Category

Create a new directory under `skills/` with a `SKILL.md`:
```bash
mkdir skills/oracle-my-new-topic
touch skills/oracle-my-new-topic/SKILL.md
```

Then update the README to include the new category.

### 4. Platform Config Contributions

If you use an AI platform not yet covered, add a config file.
See `.cursor/rules/oracle-apex.mdc` as an example of the expected format.

---

## Branch Naming

```
skill/oracle-23ai-json-duality    # New skill
fix/plsql-cursor-example          # Bug fix
platform/aider-config             # New platform config
docs/improve-readme               # Documentation
```

## Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):
```
feat: Add Oracle 23ai JSON Relational Duality skill
fix: Correct FETCH FIRST syntax for 12c compatibility
docs: Add ORDS REST service examples to APEX skill
platform: Add Aider AI configuration
```

## Code of Conduct

Be respectful, constructive, and kind. We're all here to help the Oracle community write better code with AI.

---

*Thanks for helping make ORAXIA better for everyone!*
