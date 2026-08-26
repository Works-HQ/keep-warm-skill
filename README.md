# keep-warm

A Claude Code skill that keeps your sales pipeline warm without you thinking about it.

Every week it looks at who's in your funnel, finds genuinely fresh material each contact would actually care about, and drafts a short, personal, **no-ask** share note in your voice: here's what this is, and here's why it's interesting *to you*. You review the pack, send the ones you like, and the engine tracks cadence from there.

**It never sends anything.** Draft-only is the core invariant. A run that ends with anything sent is a failed run.

## Where it came from

This skill is a fusion of three things, and the credits matter:

1. **Sam McKenna's nurture play** ([#samsales](https://samsalesconsulting.com), the "Show Me You Know Me" people). Her "Next Level Nurturing" method: when a buyer goes quiet, educate them while they navigate internal meetings, give them ammo they can forward to stakeholders, use multiple channels, and **make no ask whatsoever**. The cadence ladder in this skill (thank-you → educational shares → blind proof touch → channel switch) is her Day 1/5/12/18 play adapted to a weekly rhythm.
2. **Roman Olivera's contextual-sharing pipeline** ([Orban Labs](https://orbanlabs.com)). Roman built a tool that shares links with his contacts where every link arrives already explained: what it is, why it's interesting. His architecture principles shaped this skill: one core pipeline (roster → research → script → deliver), every stage has its own artifact so it can be checked and retried alone, evidence stays separate from the message, and **render-from-approval**: nothing goes out that a human hasn't approved.
3. **The last30days research engine** ([mvanhorn/last30days-skill](https://github.com/mvanhorn/last30days-skill), MIT). The research layer. For each contact's topics it pulls what people actually said in the last 30 days across Reddit, X, YouTube, Hacker News and the web, so the material you share is fresh and real, not evergreen filler.

Built with [Claude Code](https://claude.com/claude-code) in one overnight session, then put through a 4-round adversarial review (two independent frontier models attacking the spec, fix between rounds) before first use.

## How it works

```
roster ──► reconcile ──► cadence ──► research ──► script ──► review pack
(CRM +      (what did      (whose      (last30days   (no-ask     (you send,
 manual)     you send?      turn is     per topic +   notes in    then confirm)
             who replied?)  it?)        your content) your voice)
```

- **Two tracks.** `active` (live proposals and conversations) gets a weekly touch. `warm` (pure keep-in-touch) is due roughly monthly.
- **Reconciliation first.** The skill can't see what you do manually, so every run starts by syncing with your mailbox: which drafts actually went out, who replied. Cadence only advances on confirmed sends.
- **Guardrails.** Skips anyone touched in the last 5 days. Stops drafting after 3 unanswered touches and flags them for a human decision instead. Never re-shares a link to the same contact. Proof numbers must come from a verified proof-points file; no number, no proof touch.
- **The no-ask rule is absolute.** No meeting requests, no proposal nudges, no calendar links in a nurture touch. The test: could they read it and owe you nothing?

## Install

1. Drop `SKILL.md` into your Claude Code skills directory as `keep-warm/SKILL.md`.
2. Install [last30days](https://github.com/mvanhorn/last30days-skill) and run its setup.
3. Adapt the marked sections of SKILL.md to your stack: your CRM query (the file shows an Attio example), your mailbox, your client-context folders, your proof-points file.
4. Run `/keep-warm`. The first run bootstraps its state files and asks before assuming anything about history.

## Commands

| Command | What it does |
|---|---|
| `/keep-warm` | Full weekly pass → draft pack |
| `/keep-warm status` | Roster, cadence position, links shared |
| `/keep-warm sent <name>` | Confirm you sent a draft (the only way cadence advances) |
| `/keep-warm unsent <name>` | Revert a mistaken confirmation |
| `/keep-warm add <name> \| <company> \| <email> \| <context>` | Add a manual contact |
| `/keep-warm remove <name>` | Opt a contact out |
| `/keep-warm track <name> active\|warm` | Move between weekly and monthly tracks |

## Credits

- Nurture methodology: **Sam McKenna** and the [#samsales](https://samsalesconsulting.com) team
- Contextual-sharing architecture: **Roman Olivera**, [Orban Labs](https://orbanlabs.com)
- Research engine: **[last30days-skill](https://github.com/mvanhorn/last30days-skill)** by mvanhorn (MIT)
- Assembled by **Zac King**, [Works](https://workshq.com.au)

MIT licensed. Use it, adapt it, tell us what you changed.
