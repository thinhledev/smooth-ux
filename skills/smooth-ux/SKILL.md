---
name: smooth-ux
description: Make a web app feel responsive and alive. Every click answers (pressed, busy, done), predictable actions update optimistically and revert on failure, slow data shows content-shaped skeletons, route changes give instant feedback, and motion is cheap (transform and opacity only). Use when someone says the app feels static, dead, laggy or unresponsive, "nothing happens when I click", asks for optimistic UI, skeleton loading, loading states, button feedback, toasts, or smooth transitions, or wants a UX polish pass on a React or Next.js front end.
---

# Smooth UX

A pass that removes silent clicks from a web front end. It works in four
layers, in this order, because each later layer builds on the earlier ones:

1. **Every control answers** on press, while working, and when done.
2. **Optimistic UI** where the outcome is predictable, with a revert path.
3. **Skeletons** shaped like the content, at route level and first read.
4. **Motion** only where it is free: transform and opacity, short, and off
   under reduced motion.

Performance beats motion. If an animation could cost a frame, a layout or a
repaint on a slow phone, leave it out.

## Process

### 1. Audit (read-only, about 10 minutes)

Find the silent spots. In a React/Next app:

- `grep -rn "await api\.\|fetch(" src --include=*.tsx`: every mutation call
  site. For each, note what the user sees between click and response.
- `grep -rn "disabled={busy\|Loading…\|Loading\.\.\." src`: label-swap and
  fade-only busy states.
- Button CSS: is there an `:active` state? Does busy look the same as
  disabled (faded)? That is the "dead button" feel.
- Routes: does each route have a `loading.tsx` (Next app router) or a
  suspense fallback? If not, every navigation is a dead click.
- Existing skeletons: generic boxes or content-shaped?
- Toasts: does success ever get said when the screen does not show it?

Report the findings in 4 to 6 bullets, then ask the user the decisions that
are theirs (use AskUserQuestion):

- Design the states first, or build to existing tokens? (If the user has a
  "design UI first" rule, design first: one state sheet with buttons, toast,
  route feedback and a skeleton per main screen.)
- Delete behaviour: keep the confirm step then remove optimistically, or no
  confirm and an Undo toast (needs a delayed or reversible delete on the
  server).
- Extras: route-level skeletons and link pending, toasts for outcomes, live
  status motion on polled progress, page fade on route change.
- Scope: product app only, or marketing and auth too.

### 2. Primitives first, screens second

Build the shared pieces once, typecheck, then apply them screen by screen.
Templates are in `references/primitives.md`. Adapt them to the project's own
tokens, CSS approach and naming; never paste colours from the template.

- Motion tokens on `:root`: press 80ms, fast 150ms, base 200ms, route 180ms,
  `--ease-out: cubic-bezier(.2,.8,.2,1)`.
- `.btn:active` scale(.97) plus a press fill.
- Busy is NOT disabled: `[aria-busy="true"]` stays at full opacity with
  `cursor: progress`.
- `Button` with `busy`, `done`, `busyLabel`, `doneLabel`; holds its resting
  width while busy so neighbours do not shift.
- `useAction` hook: `{run, busy, done}`, drops double clicks, done flashes
  1.2s, rethrows failures for the caller to report.
- Toast provider at the root: success and info auto-hide (4s, pause on
  hover/focus), errors stay and can carry a "Try again" action; max three.
- Link pending (Next 15.3+ `useLinkStatus`, rendered inside the `<Link>`):
  a dot on nav items, a thin bar on card links, a spinner on button-styled
  links.
- Skeleton block class with a translated pseudo-element sweep (not an
  animated `background-position`, which repaints), plus one skeleton
  component per main screen that reuses the real layout classes.
- Page fade: a Next `template.tsx` wrapping children in an opacity-only
  fade. Opacity only: a transform on an ancestor becomes the containing
  block of every `position: fixed` dialog inside it. Two traps, both seen in
  practice: (1) never fade the first page load, it delays first contentful
  paint (opacity 0 is not a paint candidate), so gate it on a module-level
  "already painted" flag; (2) decide the fade once at mount
  (`useState(() => painted)`), because a parent template re-renders without
  remounting on nested navigation and would otherwise fade the whole shell.
- Keep any primitive used on marketing pages free of product stylesheet
  imports (e.g. a CSS-free `LinkSpinner` module), or the landing page pays
  for the product's CSS.
- Reduced motion: kill transforms and slides, but keep spinners and skeleton
  sweeps as a slow opacity fade so work still shows.

### 3. Apply per screen: the decision table

| Action kind | Treatment |
|---|---|
| Toggle, pause/resume, mark read, dismiss, reorder, rename | Optimistic. Flip now, request behind, replace with the server's value on success, revert and say why on failure. |
| Delete | Optimistic after the confirm (or with Undo if chosen). Remove at once; on failure put it back where it was and show an error toast with Try again. If the delete navigates away, hide the id in a small module-level "pending removals" set so a stale list read does not show it again. |
| Cancel or reschedule a scheduled job | Optimistic. |
| Create, publish, pay, upload, generate, anything charging credits or validated by the server | NOT optimistic. Busy (spinner and verb in -ing), then the real result. |
| Redirect away (checkout, OAuth, portal) | Busy until the page leaves; reset on `pageshow` with `persisted` (back-forward cache). |
| Save on a page that stays | Busy, then done ("Saved" with a check) for 1.2s. |
| Navigation | Link pending indicator, then the route skeleton, then the page fades in. |
| Polled progress | Ease the bar width (400ms) between polls; status chips transition colour. |

Where to report a failure: inline if the control is still on screen,
otherwise a toast. Never both. Never a toast for something the screen
already shows changing.

For large codebases, split the screens across parallel subagents with
disjoint file lists, give each the primitives' API and this table, and
forbid them from editing the shared CSS and primitives.

### 4. Verify

- Typecheck, lint, production build.
- In a browser on mock or local data: click every changed control and watch
  for pressed, busy, done; force a failure (mock error or offline) for each
  optimistic action and see the revert and message; throttle the network and
  navigate between routes to see link pending and skeletons; check a phone
  width and reduced motion.
- Performance check: no layout-animating transitions (width is fine only on
  a small progress bar), no `background-position` animations, no animation
  on scroll.

## Copy

Busy labels are the verb in -ing with an ellipsis ("Saving…"); done labels
are past tense ("Saved"). Error copy says what failed and what state things
are in now ("Could not delete the video. It is back in your library.").
