# Multi-Agent Brainstorming Pipeline — Postmortem & Reference Guide

*Based on the AIP Support Desk project, April 2026. 20-round run with 5 agents producing a video-game-framed Colab exercise design.*

---

## Architecture

**Stack:** Bash script orchestrating `claude -p` (Claude Code CLI) invocations. Each agent is a separate CLI call with its own system prompt. Agents share context through a transcript file on disk.

**Pattern:**
```
for each round:
    for each agent:
        build prompt = phase_instructions + context + agent_identity
        response = claude -p < prompt_file --system-prompt "$agent_prompt" --model sonnet
        append response to transcript file
    if rolling_summary_interval: compress transcript
    if converge_phase: check for CONVERGED keyword
generate final summary
```

**Key files produced:**
- `brainstorm_transcript.md` — full raw transcript (appended incrementally)
- `running_summary.md` — compressed rolling summary (overwritten every N rounds)
- `brainstorm_summary.md` — final design document (generated once at end)

---

## What Worked

### Phased structure (DIVERGE → DEBATE → CONVERGE)
- Prevents premature convergence. Early rounds generated genuinely creative ideas because criticism was deferred.
- The DEBATE phase produced real pushback — agents named specific proposals and said KEEP/MODIFY/CUT.
- The CONVERGE phase forced commitment to specifics instead of endless iteration.

### Role design that creates natural tension
The best lineup was:
- **Game Designer** — owns the fun, the scoring, the player experience
- **Domain Expert** (Agentic AI Expert) — guards the technical correctness of the core concept
- **Engineer** — the realist, pushes back on fragile or infeasible ideas
- **Facilitator** (Workshop Designer) — ensures the output actually works in the target context (classroom, presentation, etc.)
- **Devil's Advocate** — pokes holes, prevents groupthink, the last to say CONVERGED

The Devil's Advocate should ALWAYS be the hardest to converge. Give it structured checklists per phase so it audits systematically, not opportunistically.

### Debate rules in the shared brief
These rules made the difference between polite round-robin and genuine debate:
- "Reference other panelists BY NAME when agreeing or disagreeing"
- "No generic praise — every response must advance the design or challenge it"
- "Propose SPECIFIC alternatives, not vague suggestions"

### Shared brief (non-negotiable framing)
Embedding the core concept constraints in a SHARED_BRIEF that all agents receive prevents scope drift. Agents can debate HOW but not WHETHER on the core framing.

---

## What Went Wrong

### ARG_MAX crash on summary generation
**Problem:** After 20 rounds, the transcript was ~315KB. Passing it as a CLI argument to `claude -p "$huge_string"` exceeded macOS ARG_MAX (1MB limit including env vars).
**Fix:** Write prompts to temp files, pipe via stdin: `claude -p --model sonnet < "$prompt_file"`. NEVER embed large transcripts in shell variables passed as CLI args.

### Convergence detection regex mismatch
**Problem:** The convergence checker grepped for `--- ROUND N ---` but the transcript wrote `--- ROUND N (PHASE) ---`. Never matched.
**Fix:** Use open-ended pattern: `sed -n "/--- ROUND $round /,$p"` (note the trailing space, no closing `---`).

### Token explosion in later rounds
**Problem:** By round 15, each agent processed ~200KB of stale context to produce ~400 words. Most of the context was settled decisions from round 5.
**Fix:** Rolling summary every N rounds. Agents read compressed summary (~2000 words) + last 2 rounds of raw transcript. Cuts token cost ~60-70%.

### Agents debugging an imaginary codebase
**Problem:** Rounds 14-20 devolved into placing data structures in cells, catching NameErrors in code that didn't exist yet. This is implementation, not brainstorming.
**Recommendation:** Split into two separate brainstorms: (1) Concept brainstorm → concept doc, (2) Implementation brainstorm → build spec. Each is shorter and more focused.

---

## Optimal Configuration (recommended defaults)

```bash
MAX_ROUNDS=15          # 20 was too many; design locked by round 12-13
SUMMARY_INTERVAL=5     # Compress at rounds 5, 10, 15
RECENT_ROUNDS_WINDOW=2 # Last 2 rounds of raw transcript alongside summary
DIVERGE_END=4          # 4 rounds of wild ideas is enough
DEBATE_END=9           # 5 rounds of debate
# CONVERGE: rounds 10-15 (6 rounds to finalize)
```

### Agent count
5 is the sweet spot. 3 is too few (no real tension). 7+ means rounds take too long and late agents in each round have too much to process.

### Model
Use `sonnet` for agents (good balance of quality/speed/cost). Use `sonnet` for the rolling summary too. Don't use `opus` for brainstorming agents — the cost multiplies by agents x rounds.

### Word limit
400 words per agent response. Enough to make a substantive point, short enough to keep rounds manageable.

---

## Checklist: Setting Up a New Brainstorming Run

1. **Define the SHARED_BRIEF** — the non-negotiable framing. What's the output? Who's the audience? What constraints are fixed?
2. **Design 4-5 agent roles** that create natural tension. Always include:
   - A domain expert (guards correctness)
   - A realist/engineer (guards feasibility)
   - A user advocate (guards usability/pedagogy/experience)
   - A devil's advocate (guards quality, converges last)
   - A creative lead (owns the vision/framing)
3. **Set phase boundaries** — DIVERGE (no criticism), DEBATE (challenge everything), CONVERGE (commit to specifics).
4. **Embed debate rules** in the shared brief: name-check others, no generic praise, specific alternatives only.
5. **Give the Devil's Advocate phase-specific checklists** — different audit questions per phase.
6. **Configure rolling summary** — every 5 rounds, compress. Agents read summary + last 2 rounds.
7. **Test with a short run first** — `MAX_ROUNDS=3 SUMMARY_INTERVAL=2 ./agentic_brainstorm.sh` to verify plumbing works.
8. **Pipe prompts via stdin** — never embed large transcripts in CLI arguments.

---

## Adapting the Roles to Other Domains

The 5-role pattern generalizes. Replace the domain-specific names but keep the structural tension:

| Structural Role | This Project | Product Design Example | Research Paper Example |
|---|---|---|---|
| Creative Lead | Game Designer | UX Designer | Hypothesis Generator |
| Domain Expert | Agentic AI Expert | Product Manager | Literature Reviewer |
| Realist | AI Engineer | Backend Engineer | Methods Statistician |
| User Advocate | Workshop Facilitator | Customer Researcher | Reader/Reviewer |
| Adversary | Devil's Advocate | Devil's Advocate | Devil's Advocate |

The Devil's Advocate role is universal. Always include it. Always make it the last to converge.

---

## Cost Estimate (rough)

**With rolling summary** (sonnet, 5 agents, 15 rounds):
- Rounds 1-5: ~5 agents x 5 rounds x ~10K tokens input ≈ 250K input tokens
- Rolling summary at round 5: ~50K input + ~4K output
- Rounds 6-10: ~5 agents x 5 rounds x ~8K tokens input ≈ 200K input tokens
- Rolling summary at round 10: ~50K input + ~4K output
- Rounds 11-15: ~5 agents x 5 rounds x ~8K tokens ≈ 200K input tokens
- Final summary: ~50K input + ~8K output
- **Total: ~800K-1M input tokens, ~40K output tokens**

**Without rolling summary** (what the 20-round run actually cost):
- ~5 agents x 20 rounds x avg ~100K tokens input ≈ 10M input tokens
- **Rolling summary saves roughly 90% of input tokens in later rounds**

---

## Future Improvements (not yet implemented)

1. **Concept/Implementation split** — two separate brainstorms. First produces a concept doc, second takes it as input and produces a build spec. Prevents late-round degeneration into pseudo-debugging.
2. **"Outsider Test" round** — after convergence, a fresh agent that only sees the summary (not the transcript) reviews it as a naive end-user. Catches blind spots from 20 rounds of groupthink.
3. **Structured output suffix** — require each agent to end with `PROPOSALS: / DECISIONS: / OPEN QUESTIONS:` to make convergence detection semantic rather than keyword-based.
4. **Cost counter in terminal** — estimated token spend per round, running total. Lets you kill a run that's spinning its wheels.
5. **Parallel agent invocation** — agents in the same round could run concurrently (they read the same transcript). Would halve wall-clock time. Tricky: responses must be collected and appended in consistent order.
