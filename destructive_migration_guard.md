Got it. Here’s the clean, **fully correct** way to implement your “industry” control with your real org setup:

* **Team**: `Vitract Server` (ultimate reviewer group)
* **Kahmyl** usually opens PRs and **cannot self-approve/merge** because branch protection requires at least one reviewer.
* You still want the **diff-based destructive migration guard** + **explicit marker** + **human approval**.

Below is the updated runbook with those facts baked in.

---

# Runbook: Destructive Migration Guard + Team Review Enforcement (Vitract Server)

## Objective

Prevent destructive schema operations from landing without:

1. CI flagging them if unapproved (diff-based scan),
2. Explicit “approved” marker immediately above each destructive line, and
3. A required reviewer from **Vitract Server** approving the PR (ensures Kahmyl can’t self-merge).

---

## Scope

* Branches: `main`, `staging`, `develop`
* Migration directory: `src/common/migrations/`
* Marker rule: **must be immediately above** the destructive operation (no blank line)
* Human approval: **Vitract Server team** must approve migration changes

---

## Policy

### Destructive operations (CI fails if found in PR diff)

Case-insensitive match in added lines:

* `DROP TABLE`
* `DROP COLUMN`
* `TRUNCATE TABLE`
* `DROP CONSTRAINT`
  (Optional later: `DROP INDEX`, `DELETE FROM`)

### Approval marker format (strict)

Marker must be on the **immediately preceding line**:

```ts
// MIGRATION_APPROVED: DROP_COLUMN (ticket: DB-123, reviewer: @VitractServerMember, reason: legacy cleanup)
await queryRunner.query(`ALTER TABLE users DROP COLUMN legacy_col`);
```

Rules:

* No blank line between marker and destructive line.
* Every destructive line needs its own marker.

---

# Implementation Steps

## Step 1 — Add destructive migration diff scanner

Create: `scripts/ci/check-destructive-migrations.js`

```js
#!/usr/bin/env node
/**
 * Destructive Migration Guard (STRICT)
 * - Scans ONLY added lines in PR diff
 * - Only for files in src/common/migrations/
 * - If a destructive statement is added, it MUST have MIGRATION_APPROVED on the immediately previous line
 */
const { execSync } = require("node:child_process");

const BASE_REF = process.env.GITHUB_BASE_REF || "main";
const ALLOW_MARKER = "MIGRATION_APPROVED:";
const MIGRATION_PATH_REGEX = /^src\/common\/migrations\/.+\.(ts|js)$/i;

const destructivePatterns = [
  { name: "DROP_TABLE", re: /\bdrop\s+table\b/i },
  { name: "DROP_COLUMN", re: /\bdrop\s+column\b/i },
  { name: "TRUNCATE_TABLE", re: /\btruncate\s+table\b/i },
  { name: "DROP_CONSTRAINT", re: /\bdrop\s+constraint\b/i },
];

function sh(cmd) {
  return execSync(cmd, { stdio: ["ignore", "pipe", "pipe"] })
    .toString("utf8")
    .trim();
}

function fail(msg) {
  console.error(msg);
  process.exit(1);
}

// Determine changed files against base
let changedFiles;
try {
  changedFiles = sh(`git diff --name-only origin/${BASE_REF}...HEAD`)
    .split("\n")
    .map((s) => s.trim())
    .filter(Boolean);
} catch {
  fail(
    `Failed diff against origin/${BASE_REF}. Ensure actions/checkout uses fetch-depth: 0.`
  );
}

const migrationFiles = changedFiles.filter((f) => MIGRATION_PATH_REGEX.test(f));

if (migrationFiles.length === 0) {
  console.log("✅ No migration files changed.");
  process.exit(0);
}

// Need 1 line context for strict "immediately above" check
const diff = sh(
  `git diff -U1 origin/${BASE_REF}...HEAD -- ${migrationFiles
    .map((f) => `"${f}"`)
    .join(" ")}`
);

const lines = diff.split("\n");
const violations = [];

for (let i = 0; i < lines.length; i++) {
  const line = lines[i];

  if (line.startsWith("+++ ") || line.startsWith("--- ") || line.startsWith("@@")) continue;

  // Only added lines
  if (!line.startsWith("+") || line.startsWith("+++")) continue;

  const added = line.slice(1);

  for (const p of destructivePatterns) {
    if (!p.re.test(added)) continue;

    const prevRaw = lines[i - 1] || "";
    const prev = prevRaw.replace(/^[-+]/, "");
    const prevIsBlank = prev.trim() === "";

    const approved = !prevIsBlank && prev.includes(ALLOW_MARKER);

    if (!approved) {
      violations.push({
        type: p.name,
        line: added.trim(),
        hint: `Add immediately above: // ${ALLOW_MARKER} ${p.name} (ticket: XYZ, reviewer: @user, reason: ...)`,
      });
    }
  }
}

if (violations.length) {
  console.error("\n❌ Unapproved destructive migration statements detected.\n");
  for (const v of violations) {
    console.error(`- [${v.type}] ${v.line}`);
    console.error(`  -> ${v.hint}\n`);
  }
  process.exit(1);
}

console.log("✅ Destructive migration guard passed.");
process.exit(0);
```

(Optional) Make executable:

```bash
chmod +x scripts/ci/check-destructive-migrations.js
```

---

## Step 2 — Add GitHub Action workflow

Create: `.github/workflows/migration-guard.yaml`

```yaml
name: Migration Guard

on:
  pull_request:
    branches: [main, staging, develop]

jobs:
  destructive_migration_guard:
    name: Destructive Migration Guard
    runs-on: ubuntu-latest

    steps:
      - name: Checkout (full history for diff)
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "18"

      - name: Run guard
        run: node scripts/ci/check-destructive-migrations.js
```

---

## Step 3 — Add CODEOWNERS for migration files

Create: `.github/CODEOWNERS`

> Important: Team handles must use the format `@ORG/TEAM`.
> You said the team is **Vitract Server**. You must use the exact GitHub org + team slug.

Add:

```txt
/src/common/migrations/  @<ORG_SLUG>/<TEAM_SLUG>
```

Example (illustration only):

```txt
/src/common/migrations/  @vitract/Vitract-Server
```

This ensures any PR touching migrations requests reviews from the team.

---

## Step 4 — Branch protection rules (ensures Kahmyl cannot merge alone)

### For `main`

* Require PR before merge ✅
* Require approvals: **1** ✅
* Require Code Owner review ✅
* Require status checks to pass ✅

  * include `Migration Guard`
  * include your main CI checks
* (Recommended) Dismiss stale approvals when new commits are pushed ✅

Effect:

* Kahmyl opens PR → cannot approve his own PR → must get **someone in Vitract Server** to approve.

### For `staging`

Same as main.

### For `develop`

Same as main (or you can allow fewer restrictions, but you said you want all 3 branches enforced).

---

# Operational Flow (How devs use it)

## Non-destructive migrations

* No marker required
* Normal review flow

## Destructive migrations

1. Add marker immediately above each destructive statement
2. Open PR
3. CI passes only if marker exists correctly
4. Someone from **Vitract Server** approves
5. Merge

---

# Verification Plan

## Test A — No migration change

✅ Guard passes.

## Test B — Destructive statement without marker

❌ Guard fails and prints the exact line + fix hint.

## Test C — Marker present but not immediately above

❌ Guard fails (strict rule).

## Test D — Marker immediately above

✅ Guard passes, but still requires Vitract Server approval before merge.
