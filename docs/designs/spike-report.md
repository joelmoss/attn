# Spike report: readability and platform (T1/C1)

Run on 2026-10-30 on macOS 27.0.1 with Xcode 27.1, Swift 6.4 and Claude Code 2.1.314. It answers Next Steps #1 in [`attn.md`](attn.md). Where this report and `attn.md` disagree on a fact, this report is newer.

The throwaway app is on the local branch `spike/readability-platform` (`app/Spike`). It is not meant to merge.

## Summary

| Item | Result | Status |
|---|---|---|
| Ring legibility at Dock sizes | Readable in rendered sheets at 32, 43, 48 and 64 pt, but needs a dark tile backing | **Needs your glance test on the real Dock** |
| Dock redraw limit | 30 state changes in about 0.9 s gave 4 redraws, each 1–4 ms | Verified |
| Pill behaviour | Never activates the app or takes key status when shown or expanded | Verified, except a real click |
| Pill placement | Correct on one plain external display | **Notched display untested** |
| Carbon hotkey | Registers without Accessibility. Same-process duplicates and most macOS shortcuts can be detected, other apps' hotkeys can't | Verified; plan needs narrowing. **A real keypress is untested** |
| Claude Code hooks | All 7 events exist; ordering, exit codes and `agent_id` confirmed live | Verified, with 4 plan changes below |
| Banner permission | The app has the code path; the system prompt needs a person | **Needs you** |
| Time-sensitive banners | An ad-hoc build with the entitlement is killed at launch | **Blocked until there is a Developer ID profile** |
| GitHub `ci_activity` | No such thread on this account; the rules come from docs only | **Unverified live** |
| OV-1 | Evidence strongly favours option A | **Your decision** |

## Dock ring

Sheets render each scenario at 32, 43, 48 and 64 pt at 2x, on dark, light and mid-tone backgrounds, scaled down from the tile the way the Dock does. `NSDockTile.size` is 128 × 128.

- [Plan spec, dark backing](spike/ring-spec-backed.png)
- [Bolder variant, dark backing](spike/ring-bold-backed.png)
- [Plan spec, no backing](spike/ring-spec-bare.png)
- [Aggregate fallback](spike/ring-aggregate-backed.png)

My read of the sheets, which is not the real-Dock test:

- **Whether and how urgently:** clear at every size. A filled colour segment stands apart from outlines even at 32 pt.
- **Backing is needed.** Without one, the orange `G` almost vanishes on a light Dock and thin outlines fade on mid-tone wallpaper. A dark squircle fixes all three backgrounds, at the cost of a ring about 25% smaller.
- **The bold variant is cleaner.** At 32 pt the plan's band outline reads as a double line. A thin arc for `running` with a thicker band (15% stroke, 10 pt gap) separates the states better.
- **`+N` is unreadable** at 32 and 43 pt. It costs little, since the panel gives the count, but it shouldn't be relied on.
- **Dashed `unknown` needs 43 pt or more**, and the `seen` mark is only clear on the bold variant.
- **The aggregate fallback is very legible** but loses running/unknown detail, so it stays the fallback.

The sheets use a fixed blue, so other accent colours were not rendered. See the proposed token change under Plan changes.

## Pill

Shown on the only connected display (LG ultrawide, 3840 × 1620, no notch), it sat 6 pt below the menu bar, centred. `safeAreaInsets.top` was 0, so the notch path was not exercised. `NSApp.isActive` stayed false and the panel never became key, including while expanded.

## Carbon hotkey

| Combo | `RegisterEventHotKey` | In `CopySymbolicHotKeys` |
|---|---|---|
| ⌃⌥⌘K (free) | 0 | no |
| ⌘Space | 0 | yes |
| ⌘Tab | 0 | yes |
| ⌘⇧3 | 0 | yes |
| ⌃↑ | 0 | no |
| Same combo, same process | **-9878** | n/a |
| All of the above, held by another process | 0 | n/a |

`RegisterEventHotKey` only reports duplicates inside its own process, so the plan's "inline conflict error" can't come from it. `CopySymbolicHotKeys` can warn about most macOS shortcuts, though it missed ⌃↑, and I didn't establish why. Conflicts with other apps can't be detected.

## Claude Code hooks

Evidence comes from the [hooks reference](https://code.claude.com/docs/en/hooks.md) and from real sessions driven through a logging hook in a scratch project. The logger kept event names and ids only. Interactive runs went through tmux, and API failures came from a local stub returning 429, 401 and 500.

### Verified behaviour

- **Events:** `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Notification`, `Stop`, `StopFailure` and `SessionEnd` all exist under those names.
- **Exit codes:** exit 2 blocks `PreToolUse`, `UserPromptSubmit` and `Stop`, among others. `Notification` and `StopFailure` ignore exit codes. Always exiting 0 is correct.
- **Timeouts:** hooks block Claude until they finish. The command default is 600 s, or 30 s on `UserPromptSubmit`, and `SessionEnd` has a 1.5 s budget by default. The 200 ms bound fits inside all of these.
- **Prompt order:** `PreToolUse`, then `PermissionRequest` about 0.1 s later, then `Notification` (`permission_prompt`) about 6 s later, then `PostToolUse` after approval.
- **Second prompt:** confirmed as a real transition, because `PostToolUse` and `PreToolUse` fall between the two prompts.
- **Idle reminder:** `idle_prompt` arrived about 60 s after `Stop`, as the plan assumes.
- **Subagents:** they share the parent's `session_id`, and their tool events carry `agent_id`. Those events kept arriving during an unanswered prompt, so D3 is needed. Main-session tool events have no `agent_id`.
- **Background subagent finishing during a prompt:** the synthetic `UserPromptSubmit` waited until the prompt was answered, so it didn't clear `waiting`.
- **Closing the terminal:** fires `SessionEnd` with `reason=other`.
- **`kill -9`:** fires no `SessionEnd`, so the item falls to `unknown` after its ttl, as the plan expects.

### Notification allowlist

| Decision | Seen live | Docs only |
|---|---|---|
| Allow | `permission_prompt` | `elicitation_dialog`, `elicitation_url_dialog` |
| Exclude | `idle_prompt` | `auth_success`, `elicitation_complete`, `elicitation_response`, `quota_auto_resume_*`, `plugin_notification` |
| Exclude until seen | none | `agent_needs_input`, `agent_completed` (listed without a description) |

The app owns the allowlist, so changing it later is a one-line change.

### Plan changes

| Finding | When it happens | Cost | Recommendation |
|---|---|---|---|
| Esc or "No" at a prompt sends no event at all | Ordinary use | The ring stays lit until the next prompt, Open or Dismiss | Accept. The next prompt clears it, and the session really is waiting for you |
| Esc during a running tool sends no event | Ordinary use | Item shows `running`, then `unknown` after the ttl, unless you type again | Accept |
| `Stop` fires while a background subagent runs, then a synthetic `UserPromptSubmit` follows when it finishes | Background subagents | Item goes done → running → done with no attention raised | Accept, document it. `Stop` carries a `background_tasks` field if this ever matters |
| `StopFailure` fires in interactive sessions only. `-p` runs ended with `SessionEnd` alone on 429, 401, 500 and refused connections | Headless `claude -p` | A failed `-p` run vanishes instead of showing an error | Accept for v1. Interactive agents are the use case |
| Banner arrives about 8 s after a prompt appears (6 s from Claude, 2 s hold) | Every prompt | Slight delay, but prompts answered within 6 s never interrupt | Keep `Notification`. Don't switch to `PermissionRequest`, which would flash for prompts you answer straight away, and which skips sandbox network prompts |
| `TERM_PROGRAM` is `tmux` inside tmux, which the plan's terminal map lacks | Anyone running Claude Code in tmux | No `open` target, so Open only acks and doesn't reach the terminal | **Your call.** Accept for v1, or add a way to find the outer terminal. tmux hides it, and I didn't test one |
| **Proposed token change (overrides design 5A):** `controlAccentColor` for `normal` follows the user's accent, so an orange accent makes `normal` and `high` look alike on the ring | Anyone with an orange accent | Urgency misread on the ring, though the panel still shows a badge | **Your call.** I recommend `systemBlue` for `normal` on the ring only |

Not reproduced: a main-session tool finishing during another tool's prompt. Bash calls ran one at a time. An `Agent` call's `PostToolUse` fired at launch, before the prompt.

## Banners and time-sensitive delivery

An ad-hoc-signed copy carrying `com.apple.developer.usernotifications.time-sensitive` was killed at launch (exit 137). There are 0 signing identities on this Mac, so real time-sensitive delivery can't be tested until the Developer ID work (D6). The unentitled behaviour of `.timeSensitive` still needs the checklist below.

## GitHub

- **Polling:** `X-Poll-Interval: 60`, `Last-Modified`, `Link` pagination and a 304 on `If-Modified-Since` all confirmed live. The `gh` token's `repo` scope is enough.
- **Scale:** 709 threads over 3 months, 108 unread, so the first-run baseline matters.
- **`ci_activity`:** none on this account, nor any `assign` thread. GitHub's docs say the reason means "a GitHub Actions workflow run that you triggered was completed", that it covers successful, failed, neutral and cancelled runs, and that users can choose failed-only.
  - **Consequences:** the plan's failed-only filter is needed. CI failures only appear for runs you triggered, and only with Actions notifications on. The app can't tell "no failures" from "setting off", so first run or `attn doctor` should say so.
  - **Unverified:** the default for that setting, `subject.type`, and whether `subject.url` is ever null.
- **Unmapped reasons:** `team_mention`, `approval_requested` and `security_alert` are real reasons the plan ignores. They weren't seen on this account.

## OV-1

Should a newer `updated_at` on a mention thread raise attention again?

Aggregates from all 32 `mention` threads on this account, one account over 3 months:

| Measure | Result |
|---|---|
| Threads whose latest text doesn't mention you, reason still `mention` | 30 of 32 |
| Later comments and reviews by others | 207 |
| Of those, mentioning you | 73, all from bots |
| Later mentions from people | **0** |
| Mentions from people, all first mentions | 4 of 101 |

GitHub's docs state the behaviour outright: the reason "remains as `mention`, regardless of whether you're ever mentioned again".

The timeline API's `mentioned` event is not a usable signal: it missed 25 of the 32 threads.

**Recommendation: option A.** `review_requested` and `assign` re-raise on a newer `updated_at`. `mention` raises once and later updates refresh only. A CI thread re-raises only when a later run fails. On this data the current rule would have re-lit the ring up to 207 times for no human request, and A loses none.

Note that 97 of 101 mentions came from bots, so even first-raise attention on this account is mostly automated. This account is mainly solo repos, so busier teams may see more human repeat mentions.

## Decisions recorded

Recorded in `AGENTS.md`:

- **Build:** XcodeGen for the app, a Swift package for `core/` and `cli/`. The spike proved the XcodeGen path, including a bundle that `UNUserNotificationCenter` and the Dock tile need. The generated project is not tracked.
- **Minimum macOS:** 15, the oldest release in Apple's usual three-version support window, which also gives `Mutex`.
- **Lint and format:** SwiftLint plus the `swift-format` in the Xcode toolchain. Neither is configured yet.

The spike code was linted with default rules and has style violations. They were left alone because the code is throwaway. No rule was disabled.

## Manual checklist

Open the app:

```
! open /Users/joelmoss/dev/attn/app/Spike/build/Build/Products/Debug/AttnSpike.app
```

1. **Ring:** at small, default and large Dock sizes, step through the scenarios for `spec` and `bold`, then press **Priority change**. Pass = a glance tells you whether something needs you and how urgently. If it fails, use `aggregate`.
2. **Pill:** press **Show pill**, hover, click. Check that the frontmost app keeps focus.
3. **Notch:** with the laptop lid open, check the pill sits below the notch. Check Spaces and a full-screen app too.
4. **Hotkey:** press ⌃⌥⌘K with another app in front, then ⌘Space. `~/Library/Logs/AttnSpike.log` shows `hotkey FIRED` if the spike received it.
5. **Banners:** press **Allow banners**, then both banner buttons. The log records whether `.timeSensitive` is refused or downgraded.
6. **CI:** check whether GitHub Actions notifications are on at github.com/settings/notifications.

The doc's Assignment (a day of real hook logging to set the ttl default) is separate. It needs your real `settings.json`, so I did not install it.
