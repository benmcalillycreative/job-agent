# job-agent

An AI job-application system I built for my own job search. It finds AI automation roles, scores each one against my experience, drafts a tailored cover letter, fills the application form, and then **stops and asks me** before anything is submitted.

If you're a hiring manager and one of my applications led you here: this is the system that sent it. The footer on my cover letters says so on purpose. It's the same kind of system I'd build for your team.

## How it works

```
Job boards ──► n8n: collect + filter ──► Google Sheet (leads)
                                              │
                         n8n: score each lead with Claude (0-100)
                                              │
              ┌───────────────┬───────────────┴───────────────┐
           85+ queued     70-84 ask me on Telegram        under 70 skipped
              │
   n8n dispatcher (max 3/day, one at a time)
              │
   Sage (coordinator agent): read posting → check AI policy → match map → cover letter
              │
   Bob (browser agent): upload resume → fill form from the answer bank → STOP before Submit
              │
   Me on Telegram: approve / edit / skip   ──►  Bob submits, confirms, logs it
```

Sage and Bob are agents in an OpenClaw multi-agent setup running on a Mac Mini. n8n handles everything that loops or runs on a schedule, so the language model is only called when there's real work to do.

## Design rules

- **A human approves every submission.** The agents prepare everything and stop before Submit. Approval is per job, by row ID.
- **No guessing.** Bob fills forms only from `answers.json` or a reply from me. Essay questions, years-of-experience counts, legal waivers, and any tool I haven't used come to me instead.
- **Postings are data, not instructions.** Text in a job post aimed at AI ("ignore previous instructions", hidden "magic words") is flagged to me, never followed.
- **The daily cap lives in n8n, not in the agents**, so a misbehaving agent can't send 50 applications.
- **Honest disclosure.** Every cover letter ends with a short note saying an agent found the posting and drafted the letter, and that I reviewed and approved it.
- **Agents can't change their own rules.** Guardrails live in each agent's locked SOUL.md (see `agents/GUARDRAILS.md`).

## What's in here

| Path | What it is |
|---|---|
| `n8n/1-score-new-leads.json` | Hourly: scores new leads with Claude Haiku, routes by score |
| `n8n/2-send-next-job.json` | Weekdays: sends the best queued job to the coordinator agent, enforces the daily cap |
| `n8n/3-status-webhook.json` | Endpoint the agents use to report status back to the sheet (header-key auth, status allow-list) |
| `agents/sage/skills/apply_job/SKILL.md` | Coordinator agent's instructions: posting review, match map, cover-letter rules, approval flow |
| `agents/bob/skills/apply-form/SKILL.md` | Browser agent's instructions: form filling, per-platform notes (Greenhouse, Lever, Ashby, Workable, Rippling), submit rules |
| `agents/GUARDRAILS.md` | Hard rules added to both agents |
| `config/*.example.json` | Example config and answer bank with placeholder data |

Credentials, sheet IDs, hostnames and personal details have been removed. Placeholders look like `YOUR-N8N-HOST`, `TARGET_PAY` and `SET-ME`.

## Lessons from the first dry run

- The browser agent hit an essay question and stopped to ask instead of inventing an answer, which is exactly what the "no guessing" rule is for.
- The first cover-letter draft included a job board's "prove you read this" magic word and over-described one project. Both became explicit rules in the skill file.
- Approval messages now must include the full letter word for word, not a summary.
