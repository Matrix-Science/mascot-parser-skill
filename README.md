# Mascot Parser Skill

**Version:** 1.0.0 &nbsp;|&nbsp; **License:** GPL-2.0

An Agent Skill that teaches AI coding assistants how to write scripts against the Mascot Server **msparser** SDK (Python, Perl, Java, C#).

The skill itself is just markdown — `skills/mascot-parser/SKILL.md` plus reference files in `skills/mascot-parser/references/`. It does not contain or redistribute the licensed msparser SDK.

## What it covers

- msparser API surface across all four language bindings
- Common recipes: reading `.dat` results, search submission, server config
- Safety rules (e.g. never submit writes against the public Matrix Science server)
- Cross-language constructor / constant / import differences

## Prerequisites

You still need the msparser SDK to actually run any code the assistant writes. Download it from:
**https://www.matrixscience.com/msparser_download.html**

The skill tells the assistant where the bindings live inside the extracted SDK.

## Install — Claude Code

Pick one location depending on whether you want the skill globally or per-project.

### Windows (PowerShell)

**Globally (all projects):**
```powershell
git clone https://github.com/Matrix-Science/mascot-parser-skill.git
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills" | Out-Null
Copy-Item -Recurse mascot-parser-skill\skills\mascot-parser "$env:USERPROFILE\.claude\skills\"
```

**Per project** (run from your project's root):
```powershell
New-Item -ItemType Directory -Force .claude\skills | Out-Null
Copy-Item -Recurse "C:\path\to\mascot-parser-skill\skills\mascot-parser" .claude\skills\
```

If you'd rather not use the command line, you can simply clone or download the repo from GitHub in File Explorer and copy the `skills\mascot-parser` folder into either:
- `%USERPROFILE%\.claude\skills\` (global), or
- `<your-project>\.claude\skills\` (per-project)

### macOS / Linux (bash)

**Globally:**
```bash
git clone https://github.com/Matrix-Science/mascot-parser-skill.git
mkdir -p ~/.claude/skills
cp -r mascot-parser-skill/skills/mascot-parser ~/.claude/skills/
```

**Per project:**
```bash
mkdir -p .claude/skills
cp -r /path/to/mascot-parser-skill/skills/mascot-parser .claude/skills/
```

### After install

Restart Claude Code (or start a new session). The skill auto-loads when you mention msparser, Mascot Server, `.dat` files, peptide/protein search results, etc. You can also force it with `/skill mascot-parser`.

## Install — other AI coding tools

The skill format follows Anthropic's [Agent Skills](https://www.anthropic.com/news/skills) spec, which several tools now support:

| Tool | Supported | Install location (Windows / Unix) |
|---|---|---|
| **Claude Code** (CLI / desktop / IDE plugins) | yes | `%USERPROFILE%\.claude\skills\` or `.claude\skills\` (project) |
| **Claude API / Agent SDK** | yes | Pass via the Skills API or bundle with your agent |
| **GitHub Copilot CLI** | yes (auto-discovered from installed plugins) | Plugin's `skills\` directory |
| **Gemini CLI** | yes (loaded via `activate_skill`) | `%USERPROFILE%\.gemini\skills\` |
| **Cursor / Windsurf / Cline / Aider / etc.** | no native skill loader | See "Manual use" below |

### Manual use (any assistant)

For tools that don't yet support the skill format, the SKILL.md is just a long-form prompt. You have two options:

1. **Paste it in** — open `skills/mascot-parser/SKILL.md` and paste the contents into the chat / system prompt.
2. **Reference it as context** — most assistants let you @-mention or attach files. Point at `skills/mascot-parser/SKILL.md` and the relevant `references/*.md` for the language you're using.

The references are split by language (`perl-api.md`, `java-api.md`, `csharp-api.md`) plus `common-recipes.md`, `api-classes.md`, and `server-config.md`, so you can attach only what you need.

## Using the skill

Once installed, just ask in plain language. Examples:

- *"Write a Python script that loads results.dat and exports peptide hits with score > 30 to CSV."*
- *"In Perl, list all enzymes configured on my Mascot server."*
- *"Show me the C# equivalent of `ms_searchparams.setLICENSE(...)`."*
- *"Submit an MGF as a Mascot search using the Java bindings."*

The skill will guide the assistant on correct API names, language-specific syntax (e.g. C# `_params()` vs Python `params()`), and safety constraints.

### Credentials in your own project

The skill steers the assistant toward loading Mascot credentials from environment variables or a `.env` file (e.g. `MASCOT_USER`, `MASCOT_PASS`). **Make sure your project's `.gitignore` excludes `.env`** so you don't accidentally commit your Mascot login. A minimal entry:

```gitignore
.env
.env.local
```

## Repo layout

```
skills/mascot-parser/
  SKILL.md              # main skill (Python is the default binding)
  references/
    perl-api.md
    java-api.md
    csharp-api.md
    api-classes.md      # cross-binding class reference
    common-recipes.md
    server-config.md
    obsolete-examples.md
```

The `msparser/`, `Documentation/`, and `mascot server scripts/` directories are gitignored — they hold the licensed SDK and are not redistributed.

## License

The skill content in this repository (the markdown under `skills/` and the documentation files at the repo root) is licensed under the **GNU General Public License, version 2** (GPL-2.0). See [LICENSE](LICENSE) for the full text.

The msparser SDK itself is **not** covered by this license — it is proprietary software licensed separately by Matrix Science. You must obtain it from https://www.matrixscience.com/msparser_download.html under its own terms.
