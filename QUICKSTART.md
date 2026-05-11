# Quickstart · AI Agent Setup Guide

> **Audience**: AI coding agent (Claude Code / Cursor / Cline / 等).
> **Goal**: Install obsidian-capability-network into the user's Obsidian vault, ready to run.
> **Approach**: Read this entire file first. Execute tasks in order. Ask the user only at marked checkpoints.

---

## Pre-flight · Detect environment + locate repo

### 0a. Detect AI tool

Identify which AI tool you're running in (affects Task 5 and Task 6 branching):

- If you're Claude Code → `tool = claude_code`
- If you're in Cursor → `tool = cursor`
- If you're Cline (VS Code) → `tool = cline`
- If you're Codex (OpenAI) → `tool = codex`
- Otherwise → `tool = other`

If you can't determine, ask user:

> "Which AI tool are you using to set this up? (Claude Code / Cursor / Cline / Codex / other)"

### 0b. Locate this repo

Set `REPO_PATH` = the absolute path of the directory containing this `QUICKSTART.md`.
If unclear (e.g., you don't have a cwd concept), ask user:

> "What's the absolute path to the obsidian-capability-network repo you just cloned/downloaded?"

Verify: `[ -f "$REPO_PATH/QUICKSTART.md" ] || echo ERROR` should print nothing.

---

## Task 1 · Collect required inputs from user

Ask:

> "What's the absolute path to your Obsidian vault? (e.g., `/Users/you/Documents/MyVault`)"

Save the answer as `VAULT_PATH` and export it for the rest of the session:

```bash
read -r VAULT_PATH    # or take from user reply if running in agent mode
export VAULT_PATH
```

Verify:

```bash
[ -d "$VAULT_PATH" ] || { echo "ERROR: directory does not exist"; exit 1; }
```

If missing → tell user to create the vault first in Obsidian, then re-run setup.

---

## Task 2 · Verify dependencies

Run each check; report status; install instructions to user if missing.

```bash
which pdftotext        # required (PDF text extraction)
which pdftoppm         # required (PDF to image for OCR)
which tesseract        # optional (OCR for scanned PDFs)
```

If `pdftotext` or `pdftoppm` missing:

> "PDF tools missing. Install with:
> macOS: `brew install poppler`
> Linux: `sudo apt install poppler-utils`"

If `tesseract` missing → just note OCR will be unavailable; don't block setup.

---

## Task 3 · Create vault directory structure

```bash
mkdir -p "$VAULT_PATH"/{Concepts,Fields,Overviews,Templates,Sources}
```

Verify all 5 directories exist after.

---

## Task 4 · Install templates

Copy all 8 template files from this repo into the user's vault.

```bash
cp $REPO_PATH/templates/*.md "$VAULT_PATH/Templates/"
```

Verify: `ls "$VAULT_PATH/Templates/"` should show `overview.md`, `field.md`, `summary.md`, `concept_a.md`, `concept_b.md`, `concept_c.md`, `log.md`, `index.md`.

---

## Task 5 · Install skills

Branch by detected `tool`:

### tool == claude_code

```bash
mkdir -p ~/.claude/skills
cp -r $REPO_PATH/skills/* ~/.claude/skills/
```

Verify: `ls ~/.claude/skills/` shows `obsidian-ingest`, `obsidian-audit`, `obsidian-field-update`, `obsidian-overview-update`, `obsidian-workflow`.

Tell user to restart Claude Code so new skills are registered.

### tool == cursor

Cursor auto-loads `.mdc` files from `.cursor/rules/` in the workspace root. Convert each SKILL.md into a Cursor rule and place them in `$VAULT_PATH/.cursor/rules/` (so when user opens the vault as workspace, all 5 rules load automatically).

```bash
mkdir -p "$VAULT_PATH/.cursor/rules"
for skill_dir in $REPO_PATH/skills/*/; do
  skill_name=$(basename "$skill_dir")
  out="$VAULT_PATH/.cursor/rules/${skill_name}.mdc"
  # Prepend Cursor rule frontmatter to the SKILL.md body
  {
    echo "---"
    echo "description: ${skill_name}"
    echo "alwaysApply: true"
    echo "---"
    echo
    cat "$skill_dir/SKILL.md"
  } > "$out"
done
```

Verify: `ls "$VAULT_PATH/.cursor/rules/"` shows 5 `.mdc` files.

Tell user to open the vault folder as a Cursor workspace.

### tool == cline

Cline reads a single `.clinerules` file from the workspace root. Concatenate all 5 SKILL.md into one file in `$VAULT_PATH/.clinerules`.

```bash
out="$VAULT_PATH/.clinerules"
echo "# obsidian-capability-network · merged skills" > "$out"
for skill_dir in $REPO_PATH/skills/*/; do
  skill_name=$(basename "$skill_dir")
  echo >> "$out"
  echo "---" >> "$out"
  echo >> "$out"
  echo "# === SKILL: ${skill_name} ===" >> "$out"
  echo >> "$out"
  cat "$skill_dir/SKILL.md" >> "$out"
done
```

Verify: `wc -l "$VAULT_PATH/.clinerules"` returns ≥ 400 lines (the 5 SKILL.md files merged).

Tell user to open the vault folder as a Cline workspace.

### tool == codex

Codex reads a single `AGENTS.md` file from the project root. Same approach as Cline — merge all 5 SKILL.md into one file in `$VAULT_PATH/AGENTS.md`.

```bash
out="$VAULT_PATH/AGENTS.md"
echo "# obsidian-capability-network · agent instructions" > "$out"
echo >> "$out"
echo "Below are 5 skills. Invoke by name (e.g. /obsidian-workflow) and execute the matching section." >> "$out"
for skill_dir in $REPO_PATH/skills/*/; do
  skill_name=$(basename "$skill_dir")
  echo >> "$out"
  echo "---" >> "$out"
  echo >> "$out"
  echo "# === SKILL: ${skill_name} ===" >> "$out"
  echo >> "$out"
  cat "$skill_dir/SKILL.md" >> "$out"
done
```

Verify: `wc -l "$VAULT_PATH/AGENTS.md"` returns ≥ 400 lines (the 5 SKILL.md files merged).

Tell user to run `codex` from within the vault folder.

### tool == other

Tell user:

> "For your AI tool, manually load each SKILL.md content as part of system prompt when invoking the corresponding command."

---

## Task 6 · Place vault config

Branch by `tool`. In all cases, replace `VAULT = /absolute/path/to/your-vault` in the copied file with `$VAULT_PATH`.

### tool == claude_code

```bash
cp $REPO_PATH/vault-config.example.md "$VAULT_PATH/CLAUDE.md"
perl -pi -e "s|/absolute/path/to/your-vault|$VAULT_PATH|g" "$VAULT_PATH/CLAUDE.md"   # cross-platform
```

### tool == cursor

Cursor reads `.cursor/rules/*.mdc` (the same directory used in Task 5 for skills). Add vault-config as a 6th rule with `alwaysApply: true` so config is always in context:

```bash
out="$VAULT_PATH/.cursor/rules/vault-config.mdc"
{
  echo "---"
  echo "description: vault-config"
  echo "alwaysApply: true"
  echo "---"
  echo
  cat $REPO_PATH/vault-config.example.md
} > "$out"
perl -pi -e "s|/absolute/path/to/your-vault|$VAULT_PATH|g" "$out"
```

### tool == cline

The vault path config has already been merged into `.clinerules` as part of Task 5 (skills file). Manually add a CONFIG block at the top of `$VAULT_PATH/.clinerules`:

```bash
{
  echo "# === VAULT CONFIG ==="
  cat $REPO_PATH/vault-config.example.md
  echo
  echo "---"
  cat "$VAULT_PATH/.clinerules"
} > "$VAULT_PATH/.clinerules.new"
mv "$VAULT_PATH/.clinerules.new" "$VAULT_PATH/.clinerules"
perl -pi -e "s|/absolute/path/to/your-vault|$VAULT_PATH|g" "$VAULT_PATH/.clinerules"
```

### tool == codex

Same approach as Cline — prepend CONFIG to the merged `AGENTS.md`:

```bash
{
  echo "# === VAULT CONFIG ==="
  cat $REPO_PATH/vault-config.example.md
  echo
  echo "---"
  cat "$VAULT_PATH/AGENTS.md"
} > "$VAULT_PATH/AGENTS.md.new"
mv "$VAULT_PATH/AGENTS.md.new" "$VAULT_PATH/AGENTS.md"
perl -pi -e "s|/absolute/path/to/your-vault|$VAULT_PATH|g" "$VAULT_PATH/AGENTS.md"
```

### tool == other

Tell user:

> "Copy the content of `vault-config.example.md` (with VAULT path replaced by `$VAULT_PATH`) into your AI tool's system prompt or rules file."

---

## Task 7 · Verify install with a test run

Ask user:

> "Setup complete. Want to test it now? If yes, paste an absolute path to a PDF you want to ingest. (Or skip and try later.)"

If user provides a path, save as `MATERIAL_PATH` and invoke:

```bash
export MATERIAL_PATH=<user-provided-path>
# Claude Code (example)
/obsidian-workflow "$MATERIAL_PATH"
```

For other tools, instruct user to invoke the equivalent (e.g., ask their AI to "run the obsidian-workflow skill on $MATERIAL_PATH").

Verify after run:
- New file in `$VAULT_PATH/Sources/<source_type>/`
- New rows in `$VAULT_PATH/log.md` and `$VAULT_PATH/index.md`
- New concept cards in `$VAULT_PATH/Concepts/`

---

## Task 8 · Report completion

Output to user:

```
✅ obsidian-capability-network installed.

Vault: <VAULT_PATH>
Tool: <tool>
Templates: 8 placed
Skills: 5 installed (or referenced)

Next:
- /obsidian-workflow <path-to-material>     # ingest something
- /obsidian-audit --global                  # check vault health
- Edit <VAULT_PATH>/<config-file-per-tool> to customize paths
```

---

## Troubleshooting (read if any task above failed)

| Symptom | Likely cause | Fix |
|---|---|---|
| Skill not recognized after install | AI tool needs restart | Restart Claude Code / reload Cursor window |
| AI refuses to create files in vault | Workspace permission | Open the vault folder as workspace / grant FS access |
| `pdftotext` errors | poppler not installed | Run dependency install (Task 2) |
| `tags: Concept/B` not parsed | Frontmatter format issue | Check template wasn't corrupted; re-copy from repo |
| Card placed in wrong folder | Vault structure non-standard | Verify Task 3 directories exist with exact names |

---

*Once setup is verified, the user can hand off this repo. From here on they invoke skills directly via their AI tool.*
