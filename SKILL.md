---
name: vibecoat-guard
description: Universal regression prevention and quality assurance plugin. Use AUTOMATICALLY during development — before parallel agents (pre-flight), after each agent (verify), after each phase (regression), and after feature completion (quality gate with 13 expert agents scoring 10/10). Also use when the user says "vibecoat guard", "check for regressions", "quality gate", or runs any /vibecoat-guard command.
---

# Vibecoat Guard

## The Iron Law

```
NO CODE SHIPS WITHOUT REGRESSION CHECKS AND A FULL 13-AGENT QUALITY GATE SCORING 10/10
```

This skill combines regression prevention (4 structural layers) with comprehensive quality assurance (13 expert agents). It runs AUTOMATICALLY at the right moments during development. The user does not need to remember to invoke it.

## Performance Design — No Redundant Work

Each layer is calibrated to its frequency. Frequent checks are instant; rare checks are thorough.

| Layer | Frequency | Duration | What it does |
|-------|-----------|----------|-------------|
| **Verify** | After every agent | ~100ms | Marker check + count check only. NO build, NO tests. |
| **Pre-commit hook** | At commit time | ~35s | Build + tests + marker check. Catches compile/test failures. |
| **Regression suite** | After a phase | ~2 min | 3 parallel agents: structural + visual + functional. |
| **Quality gate** | After feature done | ~5-10 min | 13 parallel agents, iterative 10/10 convergence. |

**Key principle:** Verify does NOT duplicate the pre-commit hook. Build+test only runs once (at commit). Verify is purely string matching — it never blocks your flow.

---

## When This Skill Activates

### Automatic Triggers (the skill MUST self-activate)

| Moment | What runs | Why |
|--------|-----------|-----|
| Before dispatching parallel agents | **Pre-Flight** (Layer 1) | Prevent file conflicts |
| After any agent completes work | **Verify** (Layer 2) | Catch accidental deletions |
| After a phase/batch of agents complete | **Regression Suite** (Layer 3) | Full structural + visual + functional check |
| After feature implementation is "done" | **Quality Gate** (Layer 4) | 13 expert agents, 10/10 convergence |
| At every `git commit` | **Pre-commit hook** (Layer 5) | Block commits with missing markers |

### Manual Commands (also available)

| Command | What it does |
|---------|-------------|
| `/vibecoat-guard init` | Bootstrap on current project |
| `/vibecoat-guard pre-flight` | Conflict matrix for planned agents |
| `/vibecoat-guard verify` | Quick diff + manifest check |
| `/vibecoat-guard regression` | 3-agent regression suite |
| `/vibecoat-guard quality` | 13-agent quality gate (10/10 loop) |
| `/vibecoat-guard full` | verify → regression → quality pipeline |
| `/vibecoat-guard report` | Show last run results |

---

## Phase 0: Init (`/vibecoat-guard init`)

Run this ONCE per project. It bootstraps all configuration automatically.

### Steps:

1. **Detect project type** by reading config files:
   ```
   package.json          → Node.js (check for next, react, vue, express, etc.)
   requirements.txt      → Python (check for fastapi, django, flask)
   go.mod                → Go
   Cargo.toml            → Rust
   Gemfile               → Ruby
   composer.json         → PHP
   ```
   Categories: `web` | `api` | `cli` | `library`

2. **Analyze hotspot files** via git history:
   ```bash
   # Get all files changed in last 50 commits, sorted by frequency
   git log --pretty=format: --name-only -50 | sort | uniq -c | sort -rn | head -20
   ```
   Files with >5 changes in last 50 commits = hotspot. These are the files most likely to cause regressions.

3. **Extract feature markers** from each hotspot file:
   Read the file content and extract markers using these patterns:
   - Switch cases: `/case\s+'([^']+)':/g` → each case is a marker
   - Imports: `/import\s+\{([^}]+)\}\s+from/g` → each import is a marker
   - Array items with id: `/\{\s*id:\s*['"]([^'"]+)['"]/g`
   - Array items with name: `/\{\s*name:\s*['"]([^'"]+)['"]/g`
   - Registrations: `/register\(['"]([^'"]+)['"]/g`
   - Route paths: `/path:\s*['"]([^'"]+)['"]/g`
   - Python decorators: `/@(app|router)\.(get|post|put|delete)\(['"]([^'"]+)['"]/g`
   - Go handlers: `/HandleFunc\(['"]([^'"]+)['"]/g`

4. **Identify order-sensitive and forbidden patterns** in hotspot files:
   - **orderedMarkers**: Items that must appear in a specific sequence (e.g., sidebar nav items). Verify checks that each marker's position in the file is after the previous one.
   - **forbiddenMarkers**: Patterns that must NOT exist in the file (e.g., `borderRight` in sidebar after intentional removal). Verify flags any match.

   Example in manifest:
   ```json
   {
     "features": {
       "sidebar-nav-order": {
         "orderedMarkers": ["id: 'tokens'", "id: 'styles'", "id: 'health'"],
         "description": "These items must appear in this order"
       }
     },
     "forbiddenMarkers": [
       { "pattern": "borderRight", "reason": "Sidebar border was intentionally removed" }
     ]
   }
   ```

5. **Count structural elements** in each hotspot:
   - Number of import statements
   - Number of switch cases
   - Number of function/method definitions
   - Number of exported symbols

5. **Detect build and test commands**:
   ```
   package.json scripts.build    → npm run build
   package.json scripts.test     → npm test / npx vitest run
   Makefile with test target     → make test
   pytest.ini / pyproject.toml   → pytest
   go.mod                        → go test ./...
   Cargo.toml                    → cargo test
   ```

6. **Detect app URL** (for web projects):
   ```
   vite.config: server.port      → http://localhost:{port}
   next.config / package.json    → http://localhost:3000
   .env: PORT=                   → http://localhost:{PORT}
   ```

7. **Check Playwright MCP availability**:
   - Try to detect if `mcp__playwright__browser_navigate` is available
   - If NOT available, inform user:
     "Playwright MCP is not installed. This is required for browser tests (3 of the 13 quality agents). Would you like me to install it?"
   - If user confirms: add Playwright MCP to `~/.claude/settings.json`
   - If user declines: continue in code-only mode (document in config)

8. **Write configuration files**:

   `.vibecoat-guard/config.json`:
   ```json
   {
     "version": "1.0",
     "projectType": "web",
     "buildCommand": "npm run build",
     "testCommand": "npx vitest run",
     "appUrl": "http://localhost:3000",
     "playwrightAvailable": true,
     "hotspotThreshold": 5,
     "lastInit": "2026-03-22T18:00:00Z"
   }
   ```

   `.vibecoat-guard/feature-manifest.json`:
   ```json
   {
     "version": "1.0",
     "projectType": "web",
     "generated": "2026-03-22T18:00:00Z",
     "hotspots": [
       {
         "file": "src/components/Sidebar.tsx",
         "commitCount": 12,
         "risk": "high",
         "features": {
           "navigation-home": {
             "markers": ["case 'home':"],
             "description": "Home navigation entry"
           }
         }
       }
     ],
     "counts": {
       "src/components/Sidebar.tsx": { "imports": 8, "cases": 5 }
     }
   }
   ```

9. **Install pre-commit hook** (ask permission first):
   Append to `.git/hooks/pre-commit` (create if not exists, make executable):
   ```bash
   # --- regression-guard marker check ---
   MANIFEST=".vibecoat-guard/feature-manifest.json"
   if [ -f "$MANIFEST" ]; then
     node -e "
     const fs = require('fs');
     const m = JSON.parse(fs.readFileSync('$MANIFEST'));
     let fail = [];
     for (const h of m.hotspots || []) {
       if (!fs.existsSync(h.file)) continue;
       const c = fs.readFileSync(h.file, 'utf-8');
       for (const [feat, info] of Object.entries(h.features || {})) {
         for (const marker of (info.markers || [])) {
           if (!c.includes(marker)) fail.push(h.file + ': ' + feat + ' missing: ' + marker);
         }
       }
     }
     if (fail.length) {
       console.error('REGRESSION GUARD: feature markers missing!');
       fail.forEach(f => console.error('  ' + f));
       process.exit(1);
     }
     console.log('  Regression guard: all markers present.');
     " || exit 1
   fi
   # --- end regression-guard ---
   ```

10. **Configure Claude Code hooks** (ask permission first):
    Add to `.claude/settings.json`:
    ```json
    {
      "hooks": {
        "PostToolUse": [
          {
            "matcher": "Write|Edit",
            "hooks": [{
              "type": "command",
              "command": "node -e \"try{const m=JSON.parse(require('fs').readFileSync('.vibecoat-guard/feature-manifest.json','utf-8'));const f=process.env.CLAUDE_TOOL_INPUT_FILE_PATH||'';const h=m.hotspots||[];const hit=h.find(x=>f.endsWith(x.file));if(hit){console.log('HOTSPOT modified: '+hit.file+' — run /vibecoat-guard verify after changes')}}catch(e){}\""
            }]
          }
        ]
      }
    }
    ```

11. **Report init results** to user with summary of detected hotspots, markers, and configuration.

---

## Layer 1: Pre-Flight (`/vibecoat-guard pre-flight`)

**When:** BEFORE dispatching parallel agents. This MUST run automatically when you are about to launch 2+ agents in parallel.

### Steps:

1. **Read the implementation plan or user request** — identify what each agent will do
2. **For each planned agent**, determine which files it will likely modify:
   - Read the task description
   - Match against known project files using Glob
   - Identify hotspot files from manifest
3. **Build conflict matrix**:
   ```
   ┌──────────────────────┬─────────┬─────────┬─────────┐
   │ File                 │ Agent A │ Agent B │ Agent C │
   ├──────────────────────┼─────────┼─────────┼─────────┤
   │ src/Sidebar.tsx      │    W    │    W    │         │ ← CONFLICT
   │ src/Router.tsx       │    W    │         │    W    │ ← CONFLICT
   │ src/NewFeature.tsx   │    W    │         │         │
   │ src/OtherFeature.tsx │         │    W    │         │
   └──────────────────────┴─────────┴─────────┴─────────┘
   ```
4. **Recommendation**:
   - No conflicts on hotspot files → "Parallel execution SAFE"
   - Conflicts on hotspot files → "CONFLICT detected. Recommend sequential execution for agents touching: [files]"
   - Conflicts on non-hotspot files → "Low-risk conflict. Parallel OK but verify after each agent"
5. **Take baseline snapshot** of hotspot file counts (imports, cases, etc.) and save to `.vibecoat-guard/last-run.json`

---

## Layer 2: Post-Agent Verify (`/vibecoat-guard verify`)

**When:** AFTER each individual agent completes work. This MUST run automatically after any agent returns results that involved file modifications.

**Design principle: FAST.** This runs after every agent, so it must complete in under 1 second. No build, no tests — those happen at commit time via the pre-commit hook. Verify runs 3 types of checks:

### Three Check Types

1. **Required markers** (`markers`): Strings that MUST exist in the file. Missing = regression.
2. **Ordered markers** (`orderedMarkers`): Strings that must appear in a specific sequence. Wrong order = regression.
3. **Forbidden markers** (`forbiddenMarkers`): Strings that must NOT exist in the file. Present = regression.

### Steps:

1. **Get changed files**:
   ```bash
   git diff --name-only
   git diff --name-only --cached
   ```

2. **For each changed file that exists in the manifest**:
   - Read current file content
   - **Required**: Check ALL markers are still present. Missing = **REGRESSION**
   - **Ordered**: Check orderedMarkers appear in correct sequence (each one's file position must be after the previous). Wrong order = **REGRESSION**
   - **Forbidden**: Check forbiddenMarkers are NOT present. Found = **REGRESSION**

3. **Count structural elements** in changed hotspot files:
   - Count imports, switch cases, registrations, exports
   - Compare with counts in manifest
   - Lower count = **WARNING: structural elements removed**

4. **Report results** (~100ms total):
   ```
   VERIFY: PASS (57/57 markers, counts stable) [0.08s]
   ```
   OR:
   ```
   VERIFY: FAIL (2 regressions found) [0.09s]
   1. src/Sidebar.tsx: marker "case 'tokens':" MISSING
   2. src/Router.tsx: import count dropped from 15 to 13
   → Fix these before continuing.
   ```

5. **If FAIL**: Do NOT proceed with next agent or phase. Fix regressions first, then re-run verify.

**NOTE: Build + test is NOT part of verify.** The pre-commit hook handles build+test at commit time. Verify is intentionally fast (~100ms) so it doesn't slow down the development flow. The layers are designed to avoid redundant work:
- **Verify** = instant marker check (after every agent)
- **Pre-commit hook** = build + test + marker check (at commit time)
- **Regression suite** = 3 agents deep check (after a phase)
- **Quality gate** = 13 agents full review (after feature done)

---

## Layer 3: Regression Suite (`/vibecoat-guard regression`)

**When:** AFTER all agents in a phase/batch complete and pass verify. This runs a deeper check using 3 parallel agents.

### Steps:

1. **Read config** — determine project type, app URL, Playwright availability
2. **Dispatch 3 agents in parallel** (in a SINGLE message with 3 Agent tool calls):

### Agent 1: Structural Integrity Agent

```
You are a structural integrity checker for a software project.

## Your Task
Check that all tracked features are still intact by verifying feature markers in the codebase.

## Manifest
[Include full content of .vibecoat-guard/feature-manifest.json]

## Steps
1. For EACH hotspot file in the manifest:
   a. Read the file using the Read tool
   b. Check that EVERY marker listed for that file exists in the content
   c. Count: imports, switch cases, function definitions, exports
   d. Compare counts with the manifest's baseline counts
2. Report:
   - Total markers checked
   - Missing markers (with file + feature name)
   - Count changes (drops = regression warning)

## Output Format
STRUCTURAL INTEGRITY: [PASS/FAIL]
Markers: [X/Y present]
Count changes: [list any drops]

MISSING MARKERS:
- [file]: [feature] — [marker text]

SUMMARY: [one sentence]
```

### Agent 2: Visual Regression Agent (if Playwright available)

```
You are a visual regression tester. Use Playwright MCP to verify the application looks correct.

## App URL
[From config.json]

## Regression Checklist
[Include content of .vibecoat-guard/regression-checklist.json if exists]

## Steps
1. Navigate to the app URL using mcp__playwright__browser_navigate
2. Take a screenshot of the main page using mcp__playwright__browser_take_screenshot
3. Get accessibility snapshot using mcp__playwright__browser_snapshot
4. For each key page/route in the checklist:
   a. Navigate to the page
   b. Take screenshot
   c. Get accessibility snapshot
   d. Verify expected elements are present (text, buttons, sections)
5. Check dark mode if applicable:
   a. Toggle dark mode
   b. Take screenshot
   c. Verify theme class applied
6. Report which pages/elements are present and which are missing

## Output Format
VISUAL REGRESSION: [PASS/FAIL]
Pages checked: [X]
Elements verified: [X/Y present]

MISSING ELEMENTS:
- [page]: [expected element] not found

SCREENSHOTS: [describe what you observed]
SUMMARY: [one sentence]
```

### Agent 3: Functional Regression Agent (if Playwright available)

```
You are a functional regression tester. Use Playwright MCP to verify the application works correctly.

## App URL
[From config.json]

## Regression Checklist
[Include content of .vibecoat-guard/regression-checklist.json if exists]

## Steps
1. Navigate to app URL using mcp__playwright__browser_navigate
2. Test navigation: click through all major navigation items, verify pages load
3. Test keyboard shortcuts: Escape, Cmd+K, Tab navigation
4. Test interactive elements: buttons, toggles, dropdowns, forms
5. Test data loading: verify content appears (not just empty states)
6. Test error handling: verify graceful degradation
7. For API projects: use Bash to curl endpoints and verify responses
8. For CLI projects: use Bash to run commands and verify output

## Output Format
FUNCTIONAL REGRESSION: [PASS/FAIL]
Flows tested: [X]
Passing: [X/Y]

FAILURES:
- [flow name]: [what went wrong]

SUMMARY: [one sentence]
```

3. **Collect results** from all 3 agents
4. **Report combined result**:
   ```
   REGRESSION SUITE: PASS ✓
   - Structural: PASS (48/48 markers, counts stable)
   - Visual: PASS (10 pages checked, all elements present)
   - Functional: PASS (8 flows tested, all passing)
   ```
5. **If ANY agent reports FAIL**: List specific regressions and fix before proceeding.

### For non-web projects (no Playwright):
- Only Agent 1 (Structural) runs
- Agent 2 replaced with: API endpoint checker (curl tests) or CLI output checker
- Agent 3 replaced with: Integration test runner

---

## Layer 4: Quality Gate (`/vibecoat-guard quality`)

**When:** AFTER a feature implementation is complete. This MUST activate automatically when the word "done", "finished", "ready", or "complete" is used in the context of a feature. This also replaces the vibe-check skill — do NOT run vibe-check separately.

### Prerequisites

1. **Check project stack** — read config files to determine technologies
2. **Check Playwright MCP** — required for 3 browser-based agents
3. **Ensure dev server is running** — for web projects, verify the app is accessible
4. **Read feature files** — gather all relevant code for the agents to review
5. **Identify the URL/route** to test if web project

### The Expert Panel — 13 Agents

Dispatch ALL applicable agents in a SINGLE message with multiple Agent tool calls.

**HARD GATE: You MUST use the Agent tool to dispatch REAL sub-agents. You CANNOT mentally simulate what agents would score. You CANNOT write out scores yourself. You CANNOT use TaskCreate instead of Agent. If you catch yourself writing "The Architecture Agent would score..." — STOP. Dispatch the actual agent.**

Each agent receives this template (customized per domain):

```
You are a [DOMAIN] expert reviewing a feature implementation.

## Feature
[Description of what was built — be specific about functionality]

## Files to Review
[List ALL relevant file paths with brief description of each]

## Code Context
[Include the actual code content of key files — read them first]

## Test URL
[URL if web project, e.g., http://localhost:3000/feature-page]

## Your Domain: [DOMAIN NAME]
[Specific focus area — copy from the agent table below]

## Scoring Rubric
Score this feature 0-10 on [DOMAIN]. A 10 means PERFECT — zero issues, zero improvements possible.

For EACH issue found:
- Severity: CRITICAL / HIGH / MEDIUM / LOW
- Location: file:line or UI element
- Issue: What's wrong
- Fix: Specific, actionable fix instruction

## Output Format
SCORE: [0-10]
PASS: [YES if 10, NO if <10]

ISSUES:
1. [SEVERITY] [location] — [issue] → [fix]
2. ...

SUMMARY: [One sentence overall assessment]
```

#### Agent-Specific Instructions

**1. Architecture Agent**
Focus: Separation of concerns, component composition, data flow, state management, API design, error boundaries, scalability patterns. Check for proper abstractions, no circular dependencies, clean interfaces.

**2. Code Quality Agent**
Focus: TypeScript strictness, naming conventions, DRY principle, cyclomatic complexity, edge case handling, type safety (no `any` casts), no dead code, proper error handling, consistent style.

**3. Security Agent**
Focus: Auth checks, input validation, XSS/CSRF/injection prevention, secrets exposure (no hardcoded keys/tokens), OWASP top 10, destructive action safeguards, SSRF prevention, path traversal guards.

**4. Functional Test Agent** *(requires Playwright MCP)*
Additional instructions:
```
You MUST use Playwright MCP tools to actually test the feature in a real browser.
Test these scenarios:
1. Happy path — full user flow works end to end
2. Validation — invalid inputs show proper errors
3. Error states — network failures handled gracefully
4. Edge cases — empty states, boundary values, rapid interactions
5. Persistence — changes survive page reload
6. Navigation — back/forward browser buttons work correctly

Use these tools:
- mcp__playwright__browser_navigate — Navigate to URL
- mcp__playwright__browser_click — Click elements
- mcp__playwright__browser_type — Type in inputs
- mcp__playwright__browser_snapshot — Get accessibility snapshot
- mcp__playwright__browser_take_screenshot — Capture screenshot
- mcp__playwright__browser_wait_for — Wait for elements
```
Without Playwright: Score capped at 7/10 maximum. Review code for test coverage instead.

**5. UX Agent**
Focus: Flow logic, cognitive load, feedback clarity, error messages quality, loading states, empty states, progressive disclosure, consistency with platform conventions, discoverability.

**6. Visual Design Agent** *(requires Playwright MCP)*
Additional instructions:
```
You MUST use Playwright MCP to take screenshots and verify visual quality.
1. Take screenshot of main feature view
2. Take screenshot of each interactive state (hover, focus, active, disabled)
3. Take screenshot in dark mode (if applicable)
4. Check: spacing consistency, alignment, typography hierarchy, color usage, contrast ratios, component sizing
```
Without Playwright: Score capped at 7/10. Review CSS/styling code instead.

**7. Responsive Agent** *(requires Playwright MCP)*
Additional instructions:
```
You MUST use Playwright MCP with viewport resizing:
1. mcp__playwright__browser_resize width=375 height=812 — Mobile
2. mcp__playwright__browser_resize width=768 height=1024 — Tablet
3. mcp__playwright__browser_resize width=1280 height=800 — Desktop
Take screenshots at each size. Check: touch targets (min 44px), text readability, no horizontal scroll, proper breakpoints, navigation adaptation.
```
Without Playwright: Score capped at 7/10. Review responsive CSS/media queries instead.

**8. Accessibility Agent**
Focus: Semantic HTML, ARIA labels/roles, keyboard navigation (Tab, Escape, Enter), focus management, screen reader compatibility, color contrast (WCAG AA 4.5:1), form labels, alt text, skip links.

**9. Copy Agent**
Focus: Clarity, conciseness, consistent tone, no typos/grammar errors, helpful error messages, appropriate microcopy, action-oriented button labels, placeholder text quality.

**10. Branding Agent**
Focus: Visual identity alignment, tone of voice consistency, consistent terminology, logo/color usage per brand guidelines (if available), professional appearance.

**11. i18n Agent**
Focus: Hardcoded strings detection, RTL readiness, date/number formatting, text expansion room (strings may be 2x longer in other languages), no concatenated strings, translation-ready architecture.

**12. Performance Agent**
Focus: Bundle size impact, unnecessary re-renders (React), lazy loading opportunities, image optimization, API call efficiency, memoization usage, no N+1 queries.

**13. Business Analyst Agent**
Focus: Feature completeness vs requirements, user stories covered, acceptance criteria met, no scope creep, no missing flows, edge cases in requirements handled.

### Stack-Specific Agent Activation

| Project Type | Active Agents | Skipped Agents |
|-------------|---------------|----------------|
| Full web app | All 13 | None |
| Mobile web | All 13 (Responsive extra critical) | None |
| API backend | 1,2,3,4*,8,11,12,13 | 5,6,7,9,10 |
| CLI tool | 1,2,3,4*,12,13 | 5,6,7,8,9,10,11 |
| Library/package | 1,2,3,12,13 | 4,5,6,7,8,9,10,11 |

*Agent 4 (Functional Test) adapts: API → curl tests, CLI → command output tests

### Collecting Scores

After all agents complete, aggregate into a scorecard:

```
╔══════════════════════╦═══════╦════════╗
║ Agent                ║ Score ║ Status ║
╠══════════════════════╬═══════╬════════╣
║ Architecture         ║  9/10 ║ FAIL   ║
║ Code Quality         ║ 10/10 ║ PASS   ║
║ Security             ║ 10/10 ║ PASS   ║
║ Functional Tests     ║  8/10 ║ FAIL   ║
║ UX                   ║  9/10 ║ FAIL   ║
║ Visual Design        ║ 10/10 ║ PASS   ║
║ Responsive           ║  7/10 ║ FAIL   ║
║ Accessibility        ║ 10/10 ║ PASS   ║
║ Copy                 ║ 10/10 ║ PASS   ║
║ Branding             ║ 10/10 ║ PASS   ║
║ i18n                 ║  6/10 ║ FAIL   ║
║ Performance          ║ 10/10 ║ PASS   ║
║ Business Analyst     ║  9/10 ║ FAIL   ║
╠══════════════════════╬═══════╬════════╣
║ OVERALL              ║  FAIL ║ 8/13   ║
╚══════════════════════╩═══════╩════════╝
Iteration: 1 | Previous: N/A
```

Present scorecard + ALL issues to user.

### Fix & Iterate Loop

If ANY agent scored < 10:

1. **Compile all issues** sorted by severity: CRITICAL → HIGH → MEDIUM → LOW
2. **Fix ALL issues** — not just the easy ones
3. **VERIFY FIXES (Layer 2)** — fast marker + count check (~100ms):
   - Check manifest markers are still intact after fixes
   - Check structural counts haven't dropped
   - NO build/test here — that's the pre-commit hook's job
   - If verify FAILS → fix the regression FIRST, re-run verify
   - Only proceed to step 4 when verify PASSES
4. **Re-dispatch ALL 13 agents** (not just failed ones — fixes cause regressions in other domains)
5. **Update scorecard** with new scores and delta from previous iteration
6. **Repeat until ALL agents score 10/10**

### Stagnation Detection

Track scores across iterations in `.vibecoat-guard/scores-history.json`.

If the SAME agent gives the SAME score for 3 consecutive iterations:
1. Report to user: "Agent [X] has scored [Y]/10 for 3 iterations. The remaining issues may require a different approach or may be acceptable trade-offs."
2. List the stuck issues with full context
3. Ask user: "Continue iterating, or accept current state?"
4. If continue: Try a fundamentally different fix approach
5. If accept: Mark as accepted with documented exceptions in report

### Save Results

Write results to `.vibecoat-guard/last-run.json`:
```json
{
  "type": "quality",
  "timestamp": "2026-03-22T18:00:00Z",
  "iteration": 3,
  "scores": { "architecture": 10, "code-quality": 10, ... },
  "passed": true,
  "issues": [],
  "accepted_exceptions": []
}
```

---

## Layer 5: File Integrity (Pre-Commit Hook)

**When:** Automatically at every `git commit`. No user action needed.

The pre-commit hook installed during init checks all manifest markers. If any marker is missing from any hotspot file, the commit is BLOCKED with a clear error message showing exactly which markers are missing and in which files.

---

## Layer 6: Report (`/vibecoat-guard report`)

**When:** Anytime the user wants to see the current status.

### Steps:

1. Read `.vibecoat-guard/last-run.json`
2. Read `.vibecoat-guard/scores-history.json`
3. Read `.vibecoat-guard/feature-manifest.json`
4. Present:
   - Last run type, timestamp, and result (PASS/FAIL)
   - Quality gate scorecard (if last run was quality)
   - Manifest status: X hotspot files, Y markers tracked
   - Score trends: improving/stable/degrading per agent
   - Recommendations: what to focus on next

---

## Common Rationalizations — BLOCKED

| Excuse | Reality |
|--------|---------|
| "The feature looks good, quick check is fine" | 13 expert agents see what you miss. Run the full loop. |
| "We can fix visual/UX stuff later" | "Later" never comes. Fix now. |
| "Accessibility is nice-to-have" | It's a legal requirement. Not optional. |
| "Only the failed agents need re-checking" | Fixes cause regressions. ALL agents re-verify. |
| "We've been at this too long, let's ship" | Stagnation detection handles this. YOU don't decide to skip. |
| "9/10 is basically perfect" | 9 is not 10. Fix the remaining issue. |
| "I'll run the loop mentally" | Mental simulation is NOT execution. Dispatch real agents. |
| "Let me describe what agents would find" | Describing is not dispatching. Use the Agent tool. |
| "I'll verify after the commit" | Pre-commit hook exists for this. Verify BEFORE committing. |
| "This is just a small fix, no need for verify" | Small fixes on hotspot files cause the biggest regressions. Always verify. |
| "The manifest doesn't cover this file" | Then update the manifest. Don't skip the check. |

## Red Flags — STOP

If you catch yourself thinking any of these, STOP and follow the process:

- "I can review this myself without dispatching agents"
- "Let me just do a quick check instead of the full loop"
- "Most of these agents won't find anything"
- "The user won't notice these issues"
- "This is taking too long"
- "I already tested this manually"
- Dispatching agents one-by-one instead of in parallel
- Skipping verify between fix iterations in the quality gate

**ALL of these mean: Follow the process. No shortcuts.**

---

## Integration Notes

- **Replaces vibe-check**: This skill includes the complete vibe-check (Layer 4). Do NOT also run vibe-check separately.
- **Works with any development skill**: build-planned, build-verified, build-iterative — all trigger quality gate on completion.
- **Uses dispatching-parallel-agents**: For efficient 13-agent dispatch.
- **Uses verification-before-completion**: As final gate after quality loop passes.

## Graceful Degradation

```
Playwright available:     All 6 layers fully operational
Without Playwright:       Layers 1-3 structural only + Layer 4 code-only (10 agents, browser agents max 7/10)
No .vibecoat-guard/:    Run /vibecoat-guard init first (skill will prompt)
No git repo:              Layers 2-5 disabled (no diff, no commits). Only Layer 4 quality gate works.
```

## Quick Reference

```
/vibecoat-guard init        → Bootstrap project (once)
/vibecoat-guard pre-flight  → Before parallel agents
/vibecoat-guard verify      → After each agent
/vibecoat-guard regression  → After each phase (3 agents)
/vibecoat-guard quality     → After feature done (13 agents, 10/10)
/vibecoat-guard full        → verify + regression + quality
/vibecoat-guard report      → Show status

Automatic:
- Pre-commit hook blocks missing markers
- Claude Code hooks warn on hotspot edits
- Skill self-activates at the right moments
```
