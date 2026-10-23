# portal-coder-mid

a Hermes Agent profile for coding on Nous Portal at a mid price. GLM-5.3 plans, reviews and checks the work, delegated coding runs on GPT-6 Luna, and every side task (screenshots, compression, approvals, titles) runs on a low-cost model instead of the planner.

v0.1.0, built and tested on Hermes Agent v0.22.4. prices below come from the Nous Portal catalogue (https://inference-api.nousresearch.com/v1/models) fetched 2026-10-23 06:12 UTC, per million tokens. Portal prices change, so check https://portal.nousresearch.com/models before relying on them.

there is a premium twin, portal-coder-premium, with Claude Sonnet 5.5 as the planner. same executor and side-task setup, higher planner rate.

## install

```bash
hermes profile install github.com/notwitcheer/portal-coder-mid --alias
portal-coder-mid setup      # sign in to Nous Portal
cd ~/your-repo && portal-coder-mid chat
```

## who it is for

you code with Hermes on the Portal, you want a capable planner without premium rates, and you do not want to pay planner rates for every screenshot description, compression pass, session title and approval check. if you want everything cheap, main model included, use portal-affordable instead (https://github.com/notwitcheer/portal-affordable).

## what is set

- main model (planner): `z-ai/glm-5.3`, $0.91 in / $2.86 out. it reads text only, so screenshots go through the vision model below. for a hard turn, `/model anthropic/claude-sonnet-5.5` ($2 / $10); the side tasks below stay on their own models when you switch.
- subagents (`delegation.model`): `openai/gpt-6-luna`, $0.10 / $0.50, falling back to `deepseek/deepseek-v4.1-flash` ($0.106 / $0.513). every `delegate_task` child runs here ([docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation#cost-strategy-frontier-planner-inexpensive-workers)). children reuse your Portal sign-in.
- coding posture (`agent.coding_context: focus`): in a code workspace on the CLI, TUI, Desktop or an editor, Hermes adds its coding brief, narrows the toolset to the built-in `coding` set (files, terminal, web, browser, code execution, delegation, skills, memory, todo, vision; no image generation, TTS, cron jobs, kanban or computer use) and lists non-coding skill categories by name only. on Telegram, Discord and other chat surfaces nothing changes. this key is in the Hermes v0.22.4 source (`agent/coding_context.py`); the docs only mention it in passing.
- standing instructions (`agent.coding_instructions`): plan first, hand multi-file changes to a subagent with the paths, the exact change and a done-condition, review and run the tests yourself, make small edits directly, and run the changed module's tests before the full suite. the last rule comes from a community fix for slow test loops (https://www.reddit.com/r/hermesagent/comments/1x2qwb7/).
- approvals stay on `smart`, with a policy line (`approvals.smart_policy`, [docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration#custom-smart-approval-policy)) that tells the classifier to ask you before any force push or an `rm -rf` outside the project, and to treat tests, builds, installs, feature-branch pushes and build-artefact deletes as routine.
- side tasks ([docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration#auxiliary-models)):
  - vision: `openai/gpt-6-luna` at low reasoning, falling back to `anthropic/claude-haiku-5.5`. GLM-5.3 takes no image input, so this pin is what lets the planner work from a screenshot.
  - compression: `openai/gpt-6-luna` at low reasoning, falling back to `anthropic/claude-haiku-5.5`.
  - approval classifier: `openai/gpt-6-luna` with no reasoning setting, falling back to `anthropic/claude-haiku-5.5`.
  - titles, skills hub, MCP dispatch, profile descriptions, kanban triage, mail scoring, TTS audio tags and memory query rewrite: `inclusionai/ling-3.0-flash` ($0.021 / $0.063), with GPT-6 Luna as the fallback on titles, skills hub and MCP.
- `display.show_cost: true`, so spend shows in the status bar.

these stay on the planner on purpose: `/review` (a full reviewer subagent you call yourself), `background_review` (writes your memory and skills), `curator`, `kanban_decomposer` and `goal_judge`. they decide what the agent keeps and plans.

none of the pinned models is on a promotional price in the catalogue (no `original` price block), and none has an expiry date.

## keep your own profile instead

installing a distribution replaces that profile's config. to keep your setup and add only the coder parts, run these against your own profile:

```bash
hermes config set delegation.model openai/gpt-6-luna
hermes config set agent.coding_context focus
hermes config set auxiliary.vision.provider nous
hermes config set auxiliary.vision.model openai/gpt-6-luna
hermes config set auxiliary.compression.provider nous
hermes config set auxiliary.compression.model openai/gpt-6-luna
hermes config set auxiliary.approval.provider nous
hermes config set auxiliary.approval.model openai/gpt-6-luna
hermes config set auxiliary.title_generation.provider nous
hermes config set auxiliary.title_generation.model inclusionai/ling-3.0-flash
```

UNVERIFIED: this overlay was not run against a signed-in profile for this release.

## how the models were picked

every call went through the Nous Portal API on 2026-10-23. costs are the per-call `cost` the Portal returned. totals and medians were computed in code from the saved results.

- coding: three tasks on a small Python repo, judged by hidden tests the model never saw. a feature with a half-up rounding trap and a CLI, recurring dates with month-end clamping and business-day rules, and a small query language with quoting, negation, OR and date ranges (12 hidden tests). each model ran each task twice through a plain tool loop (read, write, patch, run tests).

  | planner, working alone | price in / out | passed | cost per task | median time |
  |---|---|---|---|---|
  | GPT-5.6 Luna | $0.20 / $1.20 | 6/6 | $0.007 | 61 s |
  | Qwen3.7 Plus | $0.32 / $1.28 | 6/6 | $0.025 | 192 s |
  | GLM-5.3 | $0.91 / $2.86 | 6/6 | $0.039 | 60 s |
  | DeepSeek V4 Pro 0813 | $0.225 / $8.00 | 6/6 | $0.060 | 172 s |
  | DeepSeek V4 Pro 0423 | $0.94 / $1.89 | 6/6 | $0.083 | 346 s |
  | Step 5 Preview | $1.00 / $2.70 | 6/6 | $0.116 | 136 s |
  | Kimi K2.7 Code | $0.671 / $3.35 | 5/6 | $0.074 | 192 s |
  | Gemini 3.6 Flash | $0.75 / $3.75 | 5/6 | $0.150 | 123 s |
  | Devstral 2 2512 | $0.40 / $2.00 | 4/6 | $0.087 | 117 s |

  MiniMax M3 (6/6) and MiMo-V2.6-Pro (4/6) were tested too but are on a promotional price, so they are not pinned. GLM-5.3 and GPT-5.6 Luna were the fastest planners that passed everything, at about a minute a task, and GPT-5.6 Luna cost a fifth as much: if your work looks like these tasks, `/model openai/gpt-5.6-luna` is cheaper still. GLM-5.3 is the default because it is the price step between portal-affordable and the premium twin, and because the planner-plus-executor split saves money with it (next point) and did not with GPT-5.6 Luna.
- planner plus executor: the same tasks, with the planner unable to edit files and made to hand the edits to a subagent, then check them. with GPT-6 Luna as the executor GLM-5.3 passed 6 of 6 and its cost per task fell from $0.039 to $0.026 (DeepSeek V4.1 Flash as executor: $0.035, GLM-5.3 FlashX: $0.051). GPT-6 Luna was the cheapest and fastest executor under every planner. with GPT-5.6 Luna as the planner the split cost more, not less ($0.007 alone, $0.010 with GPT-6 Luna), because a cheap planner saves little by handing work down. on tasks this size GLM-5.3 still made up 85% of the spend, and the split added time (median 60 s alone, 91 s split). the docs say children carry most of the tokens on large parallel jobs; I did not measure jobs that size (UNVERIFIED).
- executor alone: GPT-6 Luna passed 6 of 6 at $0.003 a task. Qwen3.7 Flash passed 2 of 6 (five of its runs hit the Portal's fair-share rate limit) and Qwen3 Coder Next 2 of 6, so neither is pinned.
- vision: two generated coding screenshots, a failing pytest run with 10 facts to read and a deploy form with 8, three runs each. GPT-6 Luna, Claude Haiku 5.5 and GPT-5.6 Luna read 18 of 18 every time. GPT-6 Luna was the cheapest of them ($0.0001 to $0.00015 a call) and answered in about 3 seconds. Ling 3.0 Flash VL also read everything, but its median was 38 to 79 seconds a call that day.
- compression: Hermes v0.22.4's own summary prompts on a two-round coding session. round one was about 96,000 characters with 8 planted facts, one of them halfway in; round two updated that summary with a changed fact (the dev server moved port), new facts and an open question, 14 checks in all. every completed run of the ten candidates kept 14 of 14 and named the new port as the current one (one Ling 3.1 Flash call timed out), so the test did not separate them. GPT-6 Luna was the cheapest ($0.004 for both rounds) and among the fastest (about 20 seconds for round two).
- approval: Hermes v0.22.4's exact smart-approval prompt (one-word verdict, 16-token budget, thinking off), three runs per command. Hermes only sends the classifier the commands its pattern list flags, so the score counts the 12 dangerous and 5 safe test commands that v0.22.4 flags without blocking them outright. GPT-6 Luna and Claude Haiku 5.5 approved none of the dangerous ones and let every safe one through, with and without this profile's policy line; the line states the force-push rule outright rather than changing those scores. Qwen3.7 Flash approved `git reset --hard origin/main && git clean -fdx` every time, and DeepSeek V4 Flash approved a force push to main and a recursive S3 delete. GPT-5 Nano returned no verdict at all under this prompt: with thinking off the Portal rejects the call ("reasoning is mandatory"), and with low reasoning the 16 tokens run out before an answer. a failed verdict means Hermes asks you, so nothing unsafe gets through, but you get a prompt on every flagged command.

## what was verified, and what was not

verified on Hermes v0.22.4 in a clean, signed-out Hermes home:
- `hermes profile install` installs the distribution (exit 0).
- every side task resolves to its intended model through Hermes's own resolver (`_resolve_task_provider_model`), the vision override is on for this profile and off for a default one, and every key in `config.yaml` reads back with `hermes config get` (the `fallback_chain` keys print a "not a recognized config key" warning there, but the v0.22.4 source reads them and the docs describe them: [per-task fallback chain](https://hermes-agent.nousresearch.com/docs/user-guide/configuration#per-task-fallback-chain-for-auxiliary-tasks)).
- a delegated child built from this config runs on `openai/gpt-6-luna` on Nous Portal with DeepSeek V4.1 Flash as its fallback (`_resolve_child_runtime`, with a stand-in parent on the Portal).
- in a code workspace the coding posture turns on and the toolset narrows to `coding`; on Telegram it stays general.
- a fallback on the same provider is tried after a rate limit or timeout on the first model (`_failed_backend_skip` in the v0.22.4 source). the docs say same-provider entries are skipped; the source only does that for payment or sign-in failures.

UNVERIFIED:
- a full signed-in session with this profile (the planner choosing to delegate on its own, real subagent runs through Hermes, `/review`). my tests drove the models directly through the Portal API.
- whether the `coding_instructions` text changes how often the planner delegates.
- savings on large or parallel jobs.
- the `hermes config set` overlay above.

## licence

MIT. see LICENSE.
