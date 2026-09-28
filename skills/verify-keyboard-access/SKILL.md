---
name: verify-keyboard-access
description: Verify every interactive control on a page is reachable with Tab, check the focus order against the expected visual order, and prove that primary actions fire from the keyboard rather than only the mouse. Use when a component was hand-tested with a mouse, a custom dropdown/modal/combobox was added, or an accessibility report flags an unreachable control.
license: Apache-2.0
metadata:
  version: 3.3.0
  homepage: https://www.reticle.sh
  repository: https://github.com/reticlehq/reticle
---

# Reachable is not the same as operable

A `<div onClick>` styled to look like a button can sit in the Tab order (plenty of component libraries hand out `tabindex="0"` for free) and still do nothing when you press Enter on it. Focus proves a control was focused, not that it works.

**Reticle** can inspect focus and exercise keyboard events in the running app. Its keyboard actions are synthetic, so they do not by themselves prove native browser Tab traversal or real keyboard activation. Not installed? `RETICLE_INSTALL_SOURCE=npx_skill npx @reticlehq/server@latest init`, then the [`install-and-verify`](https://github.com/reticlehq/reticle/blob/main/skills/install-and-verify/SKILL.md) skill.

## Read this before you start: an unhealthy session lies politely

Confirm a live session exists for the page and can render and observe:

```
reticle_session({ action: "list" })
```

In a disconnected, hidden or throttled tab, a key press can be accepted while timers, rendering and later observation do not advance, so nothing you read back is trustworthy. Get a usable context first, for example a leased tab:

```
reticle_run({ tool: "reticle_lease", args: { action: "acquire", url } })
```

Pass `refuseWhenThrottled: true` on the action so a paused tab fails loudly instead of silently doing nothing. If you cannot get a healthy session, report the run as **blocked**, not passed, and do not keep retrying against it.

## List the controls, then fix the expected order

List every interactive control before touching the keyboard, or you will only test the ones you already knew about:

```
reticle_look({ sessionId, action: "page", mode: "interactive" })
```

Before pressing Tab, write down the expected order from the page's visual layout: roughly top-to-bottom, left-to-right, plus any intentional exceptions. An order taken from the observed sequence describes what happened. It does not define what was expected.

## Walk the Tab order

```text
reticle_act({ sessionId, action: "press", args: { text: "Tab" } })
```

After each press, prove where focus went with a verdict, not a reading. Assert the control you expect at that position in the order you wrote down:

```
reticle_assert({
  sessionId,
  predicate: { kind: "element", query: { testid: "expected-control" }, state: "focused" },
})
```

A settled Tab is not a moved focus. Read `inputMode` and `focusMoved` in the press response. Under `inputMode: "synthetic"` the key event is dispatched programmatically, and browsers do not run their native focus traversal for a programmatic key event, so a failed assert with no `focusMoved` says nothing about the app. Report the Tab order as **unknown**, not failed. Native traversal needs real input (`inputMode: "real"`, available under `reticle drive`). Under real input, a control that never receives focus is a real defect, and so is focus landing somewhere other than the expected control.

Without real input you can still prove a control is _focusable_: `focus` it, then assert `state: "focused"`. That does not prove it sits in the Tab order (`tabindex="-1"` is focusable but skipped by Tab), so report it as focusable, not Tab-reachable.

Record the starting focus, each control that receives focus and in what order, any expected control that is skipped, and whether focus leaves the page.

## Exercise the primary action, not just the focus

Focus reaching a control is a precondition, not the check. Name the consequence, then activate the control with the keyboard:

reticle_act_and_wait({ sessionId, ref, action: "press", args: { text: "Enter" }, until: { kind: "element", query: { testid: "expected-result" } }, })

Reticle's `press` action dispatches a synthetic keyboard event. It does not reproduce the browser's native keyboard activation behavior. Therefore, a successful `until` proves that the application responds to that dispatched keyboard event; it does not by itself prove that a real keyboard user can activate the native control.

For custom controls whose keyboard handler is implemented in application code (for example, a custom `div` with an explicit Enter handler), a successful `reticle_act_and_wait` can provide evidence that the handler responds to the keyboard event.

For native controls such as `<button>`, checkbox, radio, or native `<select>`, do not treat a failed synthetic key press as proof of a keyboard-accessibility defect. Report the result as unknown unless another observable application signal independently proves that the required keyboard behavior works or fails.

Buttons and links conventionally use Enter. Checkboxes and native `<select>` controls conventionally use Space and arrow keys respectively. A custom-styled combobox is especially important to test because it may look identical to a native control while relying on application-level keyboard handlers.

Do not substitute a mouse click and claim that the keyboard interaction was verified.

An `until` predicate can already be true before the action. If Reticle reports `already_true`, it does not prove that the key press caused the consequence. Check that the expected consequence is absent first; if it is already present, reset the app state and start again, or report the result as inconclusive.

If the requirement is that an element becomes visible, use a predicate that requires visibility, since a hidden element still satisfies a presence check.

Load the exact fields with `reticle_tools({ names: ["reticle_act_and_wait"] })` rather than guessing.

## What to assert

1. **Every control from the initial look receives focus**, each proven by a `state: "focused"` assert.
2. **In the expected order:** the assert at step N names the control expected at step N.
3. **The primary action responds to a synthetic keyboard event**, proven by `reticle_act_and_wait({ until })` naming a consequence that was not already true. Report this as synthetic activation evidence, not proof of native keyboard accessibility.
4. **Focus is not trapped:** from each control, Tab reaches the next one or leaves the page.

Items 1 and 2 end in a verdict only when native Tab traversal is actually exercised and the resulting focus can be asserted. Under synthetic input, report them as **unknown**. Item 3 can produce a verdict about the synthetic keyboard event and its consequence, but that verdict does not establish native keyboard accessibility. Item 4 is a claim about a sequence, not a moment, and there is no general predicate for a focus trap, so report it as **observed evidence** unless the app exposes a signal that proves it. Say which is which.

## Honesty

Report each result as **verified** (Reticle returned a positive verdict for the exact condition), **failed**, **blocked** (no healthy session) or **unknown**. A verdict of `verified: "unknown"` is not a pass: it means Reticle could not tell what happened. Report it as unknown, and never weaken a check to make it pass. Observed evidence is not a verified result, `already_true` is not proof your action caused the consequence, and a key press that was dispatched but never settled is not a completed interaction.

Finish with the controls found, the expected Tab order, whether native Tab traversal was actually exercised, any observed focus transitions, any skipped controls that can be proven, the key you pressed, the exact consequence you waited for, whether the result was synthetic or native input, the verdict Reticle returned, and any throttling or timeout that limits it.

---

Capability reference: `curl https://docs.reticle.sh/capabilities.md`. Everything else: `curl https://docs.reticle.sh/llms.txt`.
