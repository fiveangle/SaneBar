# Session Handoff — SaneBar

**Date:** 2026-03-04
**Last released version:** `v2.1.20` (build `2120`)

---

## Session 57 (2026-03-04 evening)

### Done
- Checked inbox + GitHub issues post-release. No new user confirmations on v2.1.20 fixes yet.
- **Feature requests roadmap overhaul** — `marketing/feature-requests.md` fully audited against v2.1.20 codebase:
  - Cleared 12 shipped features to compact archive table
  - Removed 3 rejected items (bulk icon moves, third-party overlay detection, Intel support)
  - Added 4 new open features with verified implementation plans and breakage ratings
- **Competitive gap analysis** — researched Ice and Bartender for features SaneBar is missing. Found 4 viable gaps:
  1. Gradient tint (1/5 risk, ~1 day)
  2. New icon placement control (3/5 risk, 2-3 days)
  3. Auto-hide app menus / #103 (2/5 risk, 3-5 days) — Ice uses activation policy stealing, NO private APIs
  4. Per-profile trigger assignment (3/5 risk, 3-5 days)
- **Labeled GitHub #103** as `feature-request`
- **Bernard Le Du testimonial** — verified real quote ("I use SaneBar daily." from email Feb 11, 2026). Updated website to credit "Bernard Le Du, VVMac". Removed unverified Discord quote from all files.
- **#92 icon reset** — flowsworld returned with new finding: reset triggered when updating while MacBook disconnected from external monitor. Issue is closed but may need reopening.
- **Email #201** — DMCA reply from Discord (`copyright@discord.com`). Needs human review — legal matter, untouched.

### Open GitHub Issues
| # | Title | Status | Action Needed |
|---|-------|--------|---------------|
| 103 | Missing Option to Hide App Menus | Open | Feature request, labeled. Implementation plan in feature-requests.md |
| 102 | Second Menubar not working | Open | Responded with fix (left-click setting). Awaiting confirmation |
| 101 | Second Menu Bar not working | Open | Likely duplicate of #102 |
| 95 | Right/Left click on icon search | Open | Asked to update to 2.1.20 |
| 94 | Not possible to start hidden app | Open | Asked to update to 2.1.20 |
| 93 | Can not move items to visible | Open | Asked to update to 2.1.20 |
| 92 | Update resets icons (closed) | New comment | flowsworld: monitor-disconnect triggers reset. May need reopening |

### Email Requiring Action
- **#201** — DMCA from Discord. Legal. User must review.

### Key Files Changed
- `marketing/feature-requests.md` — full rewrite (roadmap + implementation plans)
- `docs/index.html` — Bernard Le Du testimonial attribution updated
- `.claude/projects/.../memory/MEMORY.md` — created with session findings

### Additional Finding (late session)
- **#92 icon position reset: ROOT CAUSE FOUND** — it's SaneBar's own `positionsNeedDisplayReset()` in `StatusBarController.swift`, not macOS. The function wipes positions when screen width changes >10%. Sparkle relaunch on different display config triggers it. No other app has this problem. Fix: per-display position backup (~30 lines, 1/5 risk). Full analysis in `marketing/feature-requests.md` and `/tmp/position_reset_research.md`.

### Next Session Priorities
1. **Fix #92 position reset** — root cause found, fix is ~30 lines in `StatusBarController.swift`, 1/5 risk. Strongest candidate for v2.1.21.
2. Respond to any user confirmations on #93/#94/#95/#101/#102
3. Review DMCA email #201
4. Consider implementing gradient tint (easiest feature win, 1/5 risk)
5. Website docs are still out of date (medium priority)
