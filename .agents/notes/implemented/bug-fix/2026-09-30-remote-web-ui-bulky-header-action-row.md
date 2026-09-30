# Agent Note: Portrait adaptation keeps a bulky third-party header action row in the title row

Status: implemented

## Problem

The portrait-touch adaptation layer (`packages/dsh-remote-web-ui/src/client/mobile-adapt.ts`) re-seats the official header actions — the agent-preset mode label and the background-task badge — from the title cluster into the tabs row: `body.dsh-remote-header-seated` switches the slot out of flow with injected CSS and `alignActionsText()` converges a `translate(dx, dy)` that paints it over the tabs row.

That seat was measured against two chips. A third-party plugin injects far more into the same slot: `dsh-session-manager` renders `.sm-header` into `conversation.session.header.actions` with six buttons (archive, pin, read-later, labels/notes, move to workspace, delete). Measured in the live GUI at portrait 393x851 with touch emulation, that row seated:

- `actions` reported `position: absolute` with `translate(-28px, 65px)` and `offsetParent === wSkVaW_root` — not the title cluster, because `titleRow`, `titleCluster` and the header are all `position: static`, so the absolute box resolved against the nearest positioned ancestor, the app root;
- the first five buttons landed at y=65 inside a 76 px header and `删除会话` wrapped to y=103, i.e. below the header and floating over the message stream;
- its published reserve was `--dsh-remote-header-actions-reserve: 401px`, wider than the 393 px viewport, so the tabs row capped itself at `max(0px, 100% - 401px)` and rendered 8 px wide, taking the conversation / trace / context tabs with it.

Desktop was unaffected: at 1600x900 the same session reported the actions slot `static` with the six buttons on one line at y=10.

## Decision

A slot carrying a wide third-party row keeps the desktop seat — the row stays in the title cluster, in flow, where React owns it.

- `holdsBulkyActionRow(actions)` detects `.sm-header` inside the actions slot. `seatHeaderActions()` folds it into the `seated` verdict, so the slot is not seated; `alignActionsText()` returns early for it. Both call `clearSeatOffsets(actions)`, which drops any `transform` on the node and removes `--dsh-remote-header-actions-reserve` from `<html>`, so a hot-updated page cannot keep a stale offset or a reserve that still squeezes the tabs row.
- Four CSS rules, all scoped to `body.dsh-remote-portrait` and gated by `:has(.sm-header)`, let the row occupy the title cluster instead of being painted elsewhere: the cluster wraps with a 4 px row gap, the actions slot takes `flex: 1 0 100%` with `flex-wrap: wrap` and `justify-content: flex-start`, and `.sm-headerBtn` tightens to `min-height: 26px; padding: 2px 7px; font-size: 12px`.

Measured after the change at 393x851: the actions slot is `static` with no transform, the buttons wrap to two compact rows at y=74 and y=104 inside a 166 px header, the tabs row is back to 345 px, the reserve is empty, and all six buttons hit-test as themselves. At 1600x900 every measured value — `actions` at `[354, 10, 607, 32]`, `.sm-header` at `[531, 10, 430, 32]`, all six buttons at y=10 — is byte-identical to the pre-change baseline.

## Alternatives considered

Detecting the row by width instead of by class (`actions.getBoundingClientRect().width > N`): rejected as fragile. The width depends on the locale, on the label text and on when React paints it, and a threshold that is right for one plugin is wrong for the next one. The `.sm-header` class is a stable contract the injecting plugin already publishes.

Shrinking the row until it fits the seat (icons only, or a single overflow menu): rejected because it would hide the labels that make six distinct actions discoverable, and because it changes what another plugin renders. The reported defect was placement, not density.

Capping the tabs row against a clamped reserve (`min(reserve, 40vw)`) while still seating the row: rejected. It repairs the 8 px tabs row but leaves the row painted below the header over the message stream, which is the defect the user actually saw.

Moving the node into the tabs row (the pre-v67 behaviour): already rejected by [the note that owns the seat](2026-09-09-mobile-remote-tap-and-adaptation-fixes.md) — re-parenting a React-owned host node breaks React's later `insertBefore` / `removeChild` anchors, and the badge inside it is a live React button a clone cannot keep. This change moves less, not more.

## Consequences

The adaptation layer now distinguishes two kinds of occupant in one official slot: the stock chips keep the seat that was measured for them, and a wide third-party row keeps the desktop placement. The discriminator is a class name owned by another package, so it is a coupling to `dsh-session-manager`'s markup: if that class changes, the row silently falls back to the seated behaviour and this defect returns. Detecting "wider than the seat can hold" in the aligning pass would generalise to any plugin, at the cost of a layout read on the tick that discovers the header; the class check is cheap and exact for the one plugin that ships this row today.

Two rules stay gated by `:has(.sm-header)` rather than applying to every title cluster, so the official chips' layout is untouched when they are the only occupant. `:has()` requires Chrome 105 or later; the reported device shipped ArkWeb on Chromium 144.

## Testing

`packages/dsh-remote-web-ui/tests/mobile-adapt.spec.ts` gains two specs: one asserting a slot holding `.sm-header` is never seated and never publishes a transform or a reserve, and one asserting the compact rules are scoped by `:has(.sm-header)` rather than applied to every title cluster. Both run in the existing jsdom environment alongside the seat specs, which still pass unchanged — the stock geometry keeps seating.

Live GUI verification used headless Chrome over CDP at 393x851 with `pointer: coarse`, portrait, five-point touch and the reported Huawei browser user agent: geometry read back before and after the change at both viewports, plus `elementFromPoint` hit tests on all six buttons and an overlap probe against the message region. Screenshots: [phone](../../../packages/dsh-remote-web-ui/docs/evidence-bulky-action-row-phone.png), [desktop](../../../packages/dsh-remote-web-ui/docs/evidence-bulky-action-row-desktop.png).
