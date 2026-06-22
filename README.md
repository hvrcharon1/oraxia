# 🏛️ ORAXIA — Universal Oracle AI Skills Platform

> **O**racle · **R**epository for **A**I · **X**-platform · **I**ntelligent **A**ssistance

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Oracle DB](https://img.shields.io/badge/Oracle-DB%2019c%2B-C74634?logo=oracle)](https://www.oracle.com/database/)
[![Oracle APEX](https://img.shields.io/badge/Oracle-APEX%2024%2B-C74634?logo=oracle)](https://apex.oracle.com)
[![Claude Code](https://img.shields.io/badge/Claude-Code-orange)](https://claude.ai/code)
[![Cursor](https://img.shields.io/badge/Cursor-AI%20Editor-blue)](https://cursor.sh)
[![GitHub Copilot](https://img.shields.io/badge/GitHub-Copilot-181717?logo=github)](https://github.com/features/copilot)
[![OpenCode](https://img.shields.io/badge/OpenCode-AI-green)](https://opencode.ai)

---

**ORAXIA** is an open-source collection of **natural language AI skill definitions** for Oracle Database and Oracle APEX development — designed to work seamlessly across **every major AI coding platform**.

Write less boilerplate. Ship better Oracle code. Let your AI actually know Oracle.

---

## 🌐 Supported AI Platforms

| Platform | Config File | Status |
|---|---|---|
| 🤖 Claude Code | `.claude/CLAUDE.md` | ✅ Ready |
| 🖱️ Cursor | `.cursor/rules/oracle-apex.mdc` | ✅ Ready |
| 🐙 GitHub Copilot / Codex | `.github/copilot-instructions.md` | ✅ Ready |
| 💻 VS Code (AI Extension) | `.vscode/ai-instructions.md` | ✅ Ready |
| 🔓 OpenCode | `opencode.md` | ✅ Ready |
| 🌀 Continue.dev | `.continue/config.json` | ✅ Ready |
| 🏄 Windsurf | `.windsurfrules` | ✅ Ready |

---

## 📚 Skill Categories

```
oraxia/
├── skills/
│   ├── oracle-sql/          # Core SQL, DDL, DML, TCL, DCL, Analytic Functions
│   ├── oracle-plsql/        # PL/SQL, Packages, Triggers, Cursors, Collections
│   ├── oracle-apex/         # Pages, Regions, Dynamic Actions, REST, ORDS
│   ├── oracle-dba/          # Administration, Storage, Users, Tablespaces, RMAN
│   ├── oracle-performance/  # Tuning, Indexes, Execution Plans, AWR, Hints
│   └── oracle-security/     # VPD, RLS, Encryption, Auditing, Wallet
├── platform-configs/        # Ready-to-copy config files per AI platform
└── docs/                    # Extended documentation
```

---

## 🚀 Quick Start

### Option 1 — Clone into your project
```bash
git clone https://github.com/hvrcharon1/oraxia.git .oraxia
```
Then copy the platform config for your AI tool (see `platform-configs/`).

### Option 2 — Use as a Git submodule
```bash
git submodule add https://github.com/hvrcharon1/oraxia.git .oraxia
```

### Option 3 — Use individual skill files
Copy any `SKILL.md` from the `skills/` directory and reference it in your AI platform config.

---

## 🔧 Platform Setup

### Claude Code
```bash
mkdir -p .claude && cp .oraxia/.claude/CLAUDE.md .claude/CLAUDE.md
```

### Cursor
```bash
mkdir -p .cursor/rules && cp .oraxia/.cursor/rules/oracle-apex.mdc .cursor/rules/
```

### GitHub Copilot / Codex
```bash
mkdir -p .github && cp .oraxia/.github/copilot-instructions.md .github/
```

### VS Code
```bash
mkdir -p .vscode && cp .oraxia/.vscode/ai-instructions.md .vscode/
```

### Windsurf
```bash
cp .oraxia/.windsurfrules .windsurfrules
```

---

## 🏛️ Full Skill Coverage

### Oracle Database SQL
- ✅ SELECT, JOINs, Subqueries, CTEs (`WITH` clause)
- ✅ Analytic / Window Functions (`OVER`, `PARTITION BY`, `LAG`, `LEAD`, `RANK`)
- ✅ DDL (CREATE TABLE, INDEX, SEQUENCE, SYNONYM)
- ✅ DML (INSERT, UPDATE, DELETE, MERGE)
- ✅ JSON & XML storage and querying
- ✅ Oracle 19c / 21c / 23ai features (JSON Relational Duality, SQL Domains)
- ✅ CONNECT BY hierarchical queries
- ✅ Pivot / Unpivot
- ✅ Flashback queries

### Oracle PL/SQL
- ✅ Anonymous blocks, Stored procedures, Functions
- ✅ Packages (Specification + Body)
- ✅ Triggers (Row-level, Statement-level, Instead-of)
- ✅ Cursors (Explicit, Implicit, REF CURSOR)
- ✅ Collections (Associative arrays, Nested tables, Varrays)
- ✅ Exception handling
- ✅ Bulk processing (FORALL, BULK COLLECT)
- ✅ DBMS_SCHEDULER, DBMS_OUTPUT, DBMS_PIPE, UTL_FILE, UTL_HTTP

### Oracle APEX
- ✅ Application & Page design patterns
- ✅ Region types: Classic Report, Interactive Report, Interactive Grid, Cards, Maps, Charts
- ✅ Dynamic Actions (JavaScript, PL/SQL, AJAX)
- ✅ Page Processes & Validations
- ✅ REST Data Sources & ORDS integration
- ✅ APEX_* APIs (APEX_JSON, APEX_WEB_SERVICE, APEX_MAIL, etc.)
- ✅ Universal Theme, Template Components, Faceted Search
- ✅ Authorization & Authentication schemes
- ✅ APEX AI / Generative AI integration (APEX 24.1+)
- ✅ Progressive Web App (PWA) configuration

### Oracle DBA & Administration
- ✅ Tablespace & Datafile management
- ✅ User, Role & Privilege management
- ✅ RMAN backup & recovery scripts
- ✅ Data Pump (expdp / impdp)
- ✅ Oracle Multitenant (CDB / PDB)
- ✅ Oracle RAC concepts
- ✅ Oracle Data Guard basics

### Performance Tuning
- ✅ Execution plan analysis (EXPLAIN PLAN, DBMS_XPLAN)
- ✅ Index strategies (B-Tree, Bitmap, Function-based, Composite)
- ✅ SQL hints
- ✅ AWR / ASH reports interpretation
- ✅ Optimizer statistics (DBMS_STATS)
- ✅ Partitioning strategies

### Security
- ✅ Virtual Private Database (VPD / RLS)
- ✅ Oracle Label Security
- ✅ Transparent Data Encryption (TDE)
- ✅ Unified Auditing
- ✅ Oracle Wallet
- ✅ APEX Access Control

---

## 🤝 Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

1. Fork the repository
2. Create your skill branch: `git checkout -b skill/oracle-23ai-json-duality`
3. Commit your changes: `git commit -m 'feat: Add Oracle 23ai JSON Relational Duality views skill'`
4. Push and open a Pull Request

---

## 📄 License

[MIT License](LICENSE) — free to use in any project.

---

*Built with ❤️ for the Oracle developer community · Works on every AI platform*
