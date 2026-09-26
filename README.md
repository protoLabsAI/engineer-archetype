# engineer-archetype

The **Engineer** archetype bundle for [protoAgent](https://github.com/protoLabsAI/protoAgent):
a hands-on pair-programming **navigator** for your own machine. Point it at a repo (a git
URL or a folder) and it clones or registers it, proves the toolchain, writes a short repo
card, and gives you a guided tour. Point it at a bug and it walks you through it **one
checkpoint per turn**: it reproduces, narrows, asks for *your* hypothesis first, puts the
evidence in front of you, and lets **you** write the fix. Then it reviews your diff, runs
the checks, and you commit.

## Navigator, not solver, and why

Most coding agents are solvers: you describe the problem and watch the diff appear. That's
fast, and it costs you the understanding. Two recent studies measured it:

- **Balepur et al., "(Im)Paired Programming: Coding Agents Improve Productivity but Harm
  Understanding"** ([arXiv 2607.26375](https://arxiv.org/abs/2607.26375)). Coding agents
  raised task speed but lowered comprehension and the ability to extend the code afterwards.
  Low-effort interaction (copy-paste prompts, auto-accepted edits) went with the lowest
  comprehension.
- **Anthropic, "How AI assistance impacts the formation of coding skills"**
  ([anthropic.com/research](https://www.anthropic.com/research/AI-assistance-coding-skills)).
  Developers using AI scored about 17% lower on comprehension. Those who asked conceptual
  questions, requested explanations, or debugged themselves with AI as a sounding board kept
  most of the learning; those who delegated generation and debugging did not.

So this persona is built around the patterns that preserve understanding:

| Rule | What it means in practice |
|---|---|
| **You author the change** | It never edits, writes, deletes, or commits unless your *latest* message says so ("apply it", "commit it"). It describes the fix in words, not replacement code, unless you ask. |
| **One checkpoint per turn** | One step (map, reproduce, narrow, locate, verify), the evidence, then a question. It doesn't chain steps on its own momentum. Repo setup is the exception: that's plumbing. |
| **Your hypothesis first** | Until you've offered one, it shows evidence, never the conclusion: no "there it is". Say "just tell me" and it will. |
| **Evidence with `path:line`** | Every claim cites a location or a command it actually ran and its real output. |
| **Point, don't paste** | It opens the exact lines in the console's code pane with `show_code` and a one-line note (a fact or a question, not the diagnosis). When you're about to type, `open_in_editor` puts your cursor on the line. |
| **GitHub is yours to post to** | It reads issues, PRs, diffs and CI runs freely, but never opens, comments on, labels, closes or merges anything, and never pushes, unless you ask in your latest message. By default it drafts the text for you to post. |
| **Teach the why** | A sentence or two on any non-obvious contract or quirk, so you can explain the bug yourself afterwards. |

The flow (the `debug-loop` skill): orient → reproduce → narrow → *your* hypothesis → locate
→ **you** fix → it reviews and verifies → **you** commit, with a message that explains why.
It ends with a three-bullet recap (cause, fix, how it was verified) you could say out loud.

## What's inside

| plugin | source | role |
|---|---|---|
| `engineer` | builtin (protoAgent ≥ 0.180.0) | the navigator skills: `repo-onboard` (setup + guided tour) and `debug-loop` (seven guided steps, plus an appendix on why LLM/agent apps "stop without answering"). Off for every other agent. |
| `craft` | builtin | `/code-review`, `/grill`, `/due-diligence` and friends, rituals you type |
| `friction` | builtin | harness-friction ledger: a rough edge leaves evidence instead of vanishing |
| `delegates` | builtin | `delegate_to`, for an optional coding agent, used only when you say so |
| `terminal` | [terminal-plugin](https://github.com/protoLabsAI/terminal-plugin) | **your** terminal: a real PTY shell in a console view beside chat |
| `github` | [github-plugin](https://github.com/protoLabsAI/github-plugin) | GitHub over the `gh` CLI, **read-only by default**: issues, PRs, diffs, CI status, repo files |

The persona is protoAgent's `engineer` soul preset (`config/soul-presets/engineer.md` in
core). The skills are core's in-tree `engineer` plugin. Both ship with protoAgent, and this
bundle names them, so there's one copy of each to keep correct.

### The terminal is yours

The terminal plugin gives **you** a real shell (xterm.js on a PTY) in the console, next to
the chat and the code pane. It's the driver's seat: run the app, poke at it, `git commit`.
The agent runs its own commands through `run_command`, which you approve. From
terminal-plugin v0.9.0 the agent can also **read** your terminals (`terminal_list`,
`terminal_read`), so "what's this error?" works on the output you're looking at. This
archetype sets `terminal.agent_access: read`, so the agent can't run commands in your
terminals. `run` (the plugin's own default) would let it run them in a visible "Agent" tab
without run_command's approval prompt, which is driver behaviour, not navigator behaviour.
Change it in Settings ▸ Plugins ▸ Terminal (`run` / `read` / `off`). v0.9.0 also brings split panes
(`⌘D` / `⌘⇧D`), standard `⌘C`/`⌘V`/`⌘F`/`⌘K` shortcuts, and login shells, so `~/.zprofile`
runs and PATH matches your usual terminal. The terminal's WebSocket is gated by the operator bearer, via a single-use ticket
so it also works on a fleet member behind the hub (terminal-plugin v0.5.1+; the bundle pins v0.9.0). On Windows it needs
`pywinpty` (declared with a `sys_platform == 'win32'` marker; the install dialog offers it).
macOS and Linux use the stdlib PTY and need nothing.

### GitHub: read by default, write when you decide

The github plugin gives the agent **read** tools over the `gh` CLI (issues, PRs and their
merge readiness, comments, diffs, CI status, repo files and contents). That's how it gets up
to speed on a codebase's history and reads the CI failure you're debugging. It also adds a
read-only **GitHub** view (Issues/PRs tabs over your repos), a **New issue** form in the
utility bar and ⌘K, and the user-only `/issue` command, which you type.

**Before you start:** `gh` must be installed and authenticated on the host: run
`gh auth login`, or paste a personal access token into Settings ▸ GitHub. It must also be on
the PATH the agent process sees; the desktop app passes your login-shell PATH. Without it,
the GitHub tools answer with a classified "gh missing / not authenticated" error and the
view shows a setup card; nothing else breaks. Public-repo reads at low volume work
unauthenticated.

**Writes are off** (`github.write: false`): the write tools (create/edit/close/merge issues
and PRs, comments, labels, assignees) aren't even bound. To let the agent post *when you ask
it to*, flip *Allow GitHub writes* on the set-up step at create time, or later set:

```yaml
github:
  write: true
```

(Settings ▸ GitHub ▸ "Allow write tools (this agent)"). Even then, the persona only writes
on your explicit instruction; merges are `confirm`-guarded by the plugin. There's no
`default_repo` pinned: the plugin derives its repos from the projects you onboard.

## Install

Pick **Engineer** in the new-agent picker (Fleet ▸ New agent, or the setup wizard), or:

```
python -m server plugin install https://github.com/protoLabsAI/engineer-archetype
```

**Core floor: protoAgent ≥ 0.180.0.** Bundles can't enforce a minimum version, so here's
what an older core misses:

- the `engineer` skill pack is missing (it ships in core);
- before 0.180.0 (#3618) the plugin dependency check ignored PEP 508 markers, so the
  terminal's Windows-only `pywinpty` showed as a **missing dependency** on macOS/Linux;
- the terminal and github plugins need 0.27.0 (console views);
- `show_code` / the code pane needs 0.179.0, `run_auto_approve` 0.177.0, `open_in_editor`
  0.176.0.

## First run

**The set-up step asks two optional questions** (after you pick Engineer). First, *Start in
a local repo* (a folder on the agent's machine): it's registered as a managed project the agent can reach, and the terminal
opens there. Second, *Allow GitHub writes* — whether the GitHub write tools bind (off by default; see above).
Leave the repo blank and onboard from chat instead:

> onboard github.com/owner/repo and give me the repo card

Everything else arrives as a **recommended default** you can change in Settings. A bundle's
create-time form may not prompt for the core `filesystem` and `onboarding` sections, so these
aren't asked:

| Setting | Default | Change it in |
|---|---|---|
| `onboarding.root` | `~/code`: clones land in `~/code/<repo>`; `register_local_project` accepts folders under it | Settings ▸ Capabilities ▸ Project onboarding |
| `onboarding.allow` | `["github.com/*"]`: any GitHub repo, private ones through your git credentials. Choosing this archetype is that consent; narrow it to `github.com/your-org/*` if you want less | same |
| `onboarding.write_default` | `true`, so "apply it" works when you ask | same |
| `filesystem.allow_run` / `run_requires_approval` | on / on: every command asks, except the list below | Settings ▸ Tools ▸ Filesystem |
| `filesystem.run_auto_approve` | read-only git: `git status`, `git diff`, `git log`, `git show`, `git branch --show-current` | same |
| `filesystem.code_pane` | `true`: binds `show_code` and the console's Code pane | same (Tools ▸ Filesystem ▸ Shell & filesystem tools) |
| `filesystem.editor_command` | `zed`. Use `code -g` for VS Code, `cursor -g` for Cursor, or empty to unbind `open_in_editor` | same |
| `github.write` | `false`: read tools only | Settings ▸ GitHub, or the *Allow GitHub writes* switch at create time |
| `friction.issue_repo` | empty (copy to clipboard). Point it at `protoLabsAI/protoAgent` if you file harness friction upstream, **never** at the repo you're working in | Settings ▸ Plugins (Friction) |
| `friction.escape_hatch_exempt` | `[run_command]`, the persona's main instrument, not an escape hatch | same |
| model | **unset**: uses your host's model connection | Settings ▸ Host / the agent's model picker |

### Auto-approve and "Allow for this session"

Tests and builds **execute the repo's code**, so they still ask by default. Once you trust a
repo, add the commands you run all the time:

```yaml
filesystem:
  run_auto_approve:
    - git status
    - git diff
    - git log
    - git show
    - git branch --show-current
    - npm test
    - npx tsc --noEmit
    - uv run pytest
    - mise exec -- npm test      # mise-managed repos: the agent prefixes toolchain commands
```

Entries are argv prefixes, matched token by token, exec'd without a shell; entries that are
too broad or contain shell metacharacters are dropped with a warning. For a one-off run of
commands, pick **Allow for this session** on the first approval (in Zed), or `/bypass` a
turn in the console. Permanent deletes always ask.

### A coding delegate (optional)

The navigator doesn't need one. If you want "you do it" to hand a well-specified fix to a
coding agent, register an ACP delegate in **Settings ▸ Delegates**, e.g. Claude Code via
`claude-agent-acp`. Use the **absolute** path to the binary: the agent process may not see
your shell's PATH. Set its `workdir` to your onboarding root, and name the project path in
the brief. The persona uses `delegate_to` only on your explicit instruction.

## The code pane and `show_code`

With `filesystem.code_pane: true` the console shows a read-only **Code** pane beside chat.
The agent calls `show_code(project, path, line, end_line, note)` to open an exact range
there, with a one-line note on why it matters, and leaves a chip in the transcript you can
click back to. The pane's **Diff** tab shows your working-tree changes, which is what the
review step reads. Click a file path in chat to open it in the pane; ⌘-click to open it in
your external editor.

## Zed: the Agent Panel

You can talk to this agent from Zed's Agent Panel through `protoagent-acp`, a small ACP
server that is an A2A client of the running agent. Nothing else needs installing; `uvx`
fetches it. Add to Zed's `settings.json`:

```jsonc
{
  "agent_servers": {
    "Engineer": {
      "type": "custom",
      // Desktop app: strip the frozen server's certificate env so the shim's HTTPS works
      // (Zed launched by the agent can inherit a path that no longer exists).
      "command": "/usr/bin/env",
      "args": [
        "-u", "SSL_CERT_FILE", "-u", "SSL_CERT_DIR",
        "/ABSOLUTE/PATH/TO/uvx",
        "--from", "git+https://github.com/protoLabsAI/protoAgent@vX.Y.Z#subdirectory=integrations/zed-acp",
        "protoagent-acp",
        "--url", "http://127.0.0.1:<port>",
        "--token-file", "<box>/workspaces/.fleet-token"
      ]
    }
  }
}
```

- **`vX.Y.Z`**: pin to your installed protoAgent release (the shim ships in the same repo).
- **`/ABSOLUTE/PATH/TO/uvx`** (`which uvx`): Zed doesn't inherit your shell's PATH.
- **`<port>`**: the agent's port (Fleet shows it). **`<box>`**: the desktop app's data
  directory. On macOS that's `~/Library/Application Support/studio.protolabs.protoagent`;
  write it out in full, since `~` isn't expanded. On a server (not desktop), drop the
  `/usr/bin/env -u …` wrapper and pass `--token-file` or `PROTOAGENT_TOKEN` for the
  instance's bearer, if it has one.

Then pick **Engineer** in the Agent Panel's new-thread menu. What you get:

- **Follow the agent**: tool cards with file locations jump into your files; `show_code`
  and `open_in_editor` cards are followable.
- **Allow once / Allow for this session / Deny** on each `run_command` approval.
- **Send Now** on a queued message steers the running turn.
- **Thread history** lists every chat the agent has, console chats included, and reopening
  one continues the same session.
- **Continue in Zed**: press it in the console (or let the agent run `open_in_editor`), then
  start a thread in Zed within 2 minutes. The new thread *is* that console chat, with its
  history replayed. It never talks over a turn that's still running in the console.

Details and limits: protoAgent's `integrations/zed-acp/README.md` and ADR 0111.

## Friction

`friction` records the harness's rough edges (a missing tool, a confusing error) in the
agent's ledger, and open friction shows up in its working state so it stops re-reporting
the same thing. Browse it with `/friction` or the Friction view. Leave `issue_repo` empty
unless you file harness issues upstream; the ledger is about protoAgent, not about the code
you're debugging.

## Pin lifecycle (ADR 0049)

The one external member (`terminal`) is pinned to a release tag, which is a **floor**:
installs take its newest *compatible* release (caret semantics; for 0.x the minor is the
boundary). `scripts/verify_bundle.py`, run by `.github/workflows/verify-bundle.yml` on
every PR and weekly, installs this manifest into a scratch agent on a fresh protoAgent
checkout, loads every member with the recommended config, and probes each declared console
view. `scripts/check_bundle_updates.py` opens a bump PR only for an out-of-range release.
### Pin-bump PR lifecycle

The `bump` job (weekly, on dispatch, and on a member's `member-released` dispatch) reuses
**one** `bump-pins` branch and PR per repo instead of piling up dated branches, and it
rewrites that branch wholesale each run, so don't hand-edit it. A PR opened with the
repository `GITHUB_TOKEN` never auto-starts its `pull_request` run: GitHub holds it as
`action_required` until a maintainer approves it. The job detects that stall, labels and
comments on the PR, and fails, so an unapproved candidate turns the schedule red instead of
rotting. Approve the run, let `verify` go green, then merge.

While this bundle depends on core that hasn't merged yet, the repo variable
`PROTOAGENT_REF` (e.g. `refs/pull/<n>/head`) points `verify` at that core ref. Delete it
once the core change lands, so `verify` tracks `main` again.

Run the verify locally from a protoAgent checkout:

```
uv run --no-sync python /path/to/engineer-archetype/scripts/verify_bundle.py /path/to/engineer-archetype
```
