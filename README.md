# Test Automation — Claude Code Setup

This project uses **Claude Code** to assist with writing and maintaining C# + Playwright end-to-end tests. The conventions Claude must follow live in [`CLAUDE.md`](./CLAUDE.md).

---

## 1. Keep Claude config global (do NOT commit)

`CLAUDE.md` and the `.claude/` directory are **local-only** in this project — they are personal/IDE configuration, not shared source. Keep them out of version control.

### Add to `.gitignore` global

```gitignore
# Tạo file (dùng ~/.gitignore_global theo chuẩn phổ biến)
touch ~/.gitignore_global

# Khai báo cho Git dùng file này global
git config --global core.excludesfile ~/.gitignore_global

cat >> ~/.gitignore_global << 'EOF'

# Claude Code — local only, never commit (global)
.claude/
CLAUDE.md
CLAUDE.local.md
EOF
```

Check
```bash
git config --global core.excludesfile        # phải in ra đường dẫn ~/.gitignore_global
git check-ignore -v CLAUDE.md                 # chạy trong 1 repo bất kỳ để xác nhận
```


The files stay on your disk; Git just no longer tracks them.

### Verify they are ignored

```bash
git check-ignore -v CLAUDE.md .claude
git status --ignored
```

> **Note:** If your team *wants* shared conventions, commit `CLAUDE.md` and instead ignore only personal overrides via `CLAUDE.local.md` and `.claude/settings.local.json`. For this project the requirement is **local-only**, so ignore both.

---

## 2. Recommended Claude Code skills / plugins for automation testing

These extend Claude Code so it generates better, more consistent tests. Install only what you trust — review each repo first.

### Playwright MCP (highest impact)
Gives Claude **live browser access** — real DOM inspection and real selector capture — instead of guessing selectors from the prompt. Prompt-only generation tends to produce plausible-looking scripts that fail at runtime; MCP lets Claude see actual page state.

```bash
claude mcp add playwright npx @playwright/mcp@latest
```

Or via the plugin marketplace:

```bash
/plugin install playwright@claude-plugins-official
```

### Playwright E2E plugin (Page Object Model + agents)
Production-grade testing toolkit: generate tests from specs, build Page Object Models, debug failures, and fix flaky tests. Ships three specialized agents — **planner** (explores the feature and maps scenarios), **generator** (turns the plan into scripts), and **healer** (repairs tests after UI changes). Enforces best practices like stable locators and AAA structure.

Useful slash commands it provides:
- Generate an E2E test from a natural-language description
- Create a Page Object Model for a page/component
- Debug a failing test from errors, screenshots, and traces
- Analyze and fix a flaky (intermittently failing) test

### Playwright browser-automation skill (model-invoked)
A skill where Claude autonomously writes and runs Playwright automation on the fly for quick validation and exploration ("test the homepage", "verify the signup flow", "screenshot mobile + desktop"). Good for ad-hoc checks; pair it with the conventions in `CLAUDE.md` so output matches the codebase.

```bash
/plugin marketplace add lackeyjb/playwright-skill
/plugin install playwright-skill@playwright-skill
```

### Language-expert skill packs
Skill packs bundling per-language experts (including **C#**) plus backend/frontend framework guidance. Helpful for keeping generated test code idiomatic and aligned with C# clean-code rules.

---

## 3. How it fits together

```
CLAUDE.md          → the rules Claude must follow (locators, waits, clean code, English-only)
Playwright MCP     → lets Claude SEE the real app (accurate selectors)
E2E plugin agents  → plan → generate → heal workflow, POM-first
Custom skills      → encode YOUR locator strategy, assertion + wait rules, framework layout
```

The custom skill / `CLAUDE.md` layer is what makes agent output fit *this* team's standards. Without it, generated coverage is technically correct but may ignore your structure, naming, and stability rules.

---

## 4. Quick start

```bash
# 1. Restore + install browsers
dotnet restore
pwsh bin/Debug/net8.0/playwright.ps1 install

# 2. Run tests
dotnet test

# 3. (CI) headless, sharded, with trace on first retry
dotnet test --settings .runsettings
```

> **Reminder:** Anything Claude writes must follow [`CLAUDE.md`](./CLAUDE.md) — semantic locators, web-first assertions, no `Sleep`, clean code, and **English only** in all code and comments.

---

> ⚠️ Trust each plugin/skill before installing. Anthropic does not control third-party plugins or the MCP servers, files, and software they include, and cannot verify behavior. Review the source repository first.
