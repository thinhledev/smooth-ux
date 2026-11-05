# Smooth UX primitives (React and Next.js templates)

Reference implementations. Adapt names, tokens and the CSS approach (CSS
modules, Tailwind, plain CSS) to the target project. Colours here are
placeholders: map them to the project's design tokens, never paste them as is.

## CSS: tokens, press, busy, spinner, skeleton, page fade

```css
:root {
  --t-press: 80ms;
  --t-fast: 150ms;
  --t-base: 200ms;
  --t-route: 180ms;
  --ease-out: cubic-bezier(0.2, 0.8, 0.2, 1);
}

.btn {
  transition:
    background-color var(--t-fast) ease,
    border-color var(--t-fast) ease,
    color var(--t-fast) ease,
    transform 120ms var(--ease-out);
}
/* Pressed: answers the moment it is touched, before any handler runs. */
.btn:active:not(:disabled):not([aria-busy="true"]) {
  transform: scale(0.97);
  transition-duration: var(--t-press);
}
.btn:disabled { opacity: 0.45; cursor: not-allowed; }
/* Busy is not disabled: full strength, progress cursor. */
.btn[aria-busy="true"],
.btn[aria-busy="true"]:disabled { opacity: 1; cursor: progress; }

.spin { width: 15px; height: 15px; flex-shrink: 0; animation: spin 0.7s linear infinite; }
@keyframes spin { to { transform: rotate(360deg); } }

/* The check that replaces a spinner when the work is done. */
.pop { animation: pop var(--t-base) var(--ease-out); }
@keyframes pop { from { opacity: 0; transform: scale(0.6); } to { opacity: 1; transform: none; } }

/* Something that has just arrived: an inline error, a notice, a new row. */
.enter { animation: enter var(--t-base) var(--ease-out); }
@keyframes enter { from { opacity: 0; transform: translateY(4px); } to { opacity: 1; transform: none; } }

/* Skeleton block. The sweep is a translated pseudo-element, never an animated
   background-position (that repaints every frame). */
.sk { position: relative; overflow: hidden; background: var(--skeleton-bg, #e8e4dc); border-radius: 6px; }
.sk::after {
  content: ""; position: absolute; inset: 0; transform: translateX(-100%);
  background: linear-gradient(90deg, transparent, rgba(255, 255, 255, 0.5), transparent);
  animation: sweep 1.4s ease-in-out infinite;
}
@keyframes sweep { to { transform: translateX(100%); } }

/* Page fade. Opacity only: a transformed ancestor would become the
   containing block of every position: fixed dialog inside it. */
.route-in { animation: route-in var(--t-route) ease-out; }
@keyframes route-in { from { opacity: 0; } }

@keyframes breathe { 50% { opacity: 0.45; } }

@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
  /* Work in progress must still show: movement becomes a slow fade. */
  .spin, .sk::after { animation: breathe 1.6s ease-in-out infinite !important; }
  .sk::after { transform: none !important; }
}
```

CSS modules scope `@keyframes` names: an animation inside a `.module.css`
cannot use a keyframe defined in global CSS. Define a local copy there.

## Button.tsx

```tsx
"use client";

import { useLayoutEffect, useRef } from "react";

export function Spinner({ size = 15 }: { size?: number }) {
  return (
    <svg className="spin" width={size} height={size} viewBox="0 0 16 16" fill="none" aria-hidden="true">
      <circle cx="8" cy="8" r="6" stroke="currentColor" strokeOpacity="0.3" strokeWidth="2" />
      <path d="M14 8a6 6 0 0 0-6-6" stroke="currentColor" strokeWidth="2" strokeLinecap="round" />
    </svg>
  );
}

export function DoneCheck({ size = 15 }: { size?: number }) {
  return (
    <svg className="pop" width={size} height={size} viewBox="0 0 16 16" fill="none" aria-hidden="true">
      <path d="m3.5 8.4 3 3 6-6.6" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" />
    </svg>
  );
}

type ButtonProps = React.ButtonHTMLAttributes<HTMLButtonElement> & {
  busy?: boolean;
  done?: boolean;
  busyLabel?: React.ReactNode;
  doneLabel?: React.ReactNode;
};

/**
 * A button that always answers: pressed on touch (CSS), busy while its work
 * runs, done when it lands. The width is held at its resting size while busy
 * or done, so "Save" turning into "Saving…" does not shove its neighbours.
 * The lock is written to the element directly: a state update for it would
 * cost a second render on every click.
 */
export function Button({ busy = false, done = false, busyLabel, doneLabel, disabled, children, type = "button", ...rest }: ButtonProps) {
  const ref = useRef<HTMLButtonElement>(null);
  const restWidth = useRef(0);
  const held = busy || done;

  useLayoutEffect(() => {
    const el = ref.current;
    if (!el) return;
    if (held) {
      if (restWidth.current) el.style.minWidth = `${restWidth.current}px`;
    } else {
      el.style.minWidth = "";
      restWidth.current = el.offsetWidth;
    }
  }, [held]);

  return (
    <button ref={ref} type={type} disabled={disabled || busy} aria-busy={busy || undefined} {...rest}>
      {busy ? (<><Spinner />{busyLabel ?? children}</>) : done ? (<><DoneCheck />{doneLabel ?? children}</>) : children}
    </button>
  );
}
```

## useAction.ts

```ts
"use client";

import { useCallback, useEffect, useRef, useState } from "react";

/**
 * Runs one async action for one control: `busy` while it runs, `done` for a
 * moment after, then rest. A second run while one is in flight is dropped.
 * A failure resets to rest at once and rethrows, so the caller reports it.
 */
export function useAction<A extends unknown[], R>(action: (...args: A) => Promise<R>, { doneMs = 1200 } = {}) {
  const [state, setState] = useState<"idle" | "busy" | "done">("idle");
  const actionRef = useRef(action);
  const running = useRef(false);
  const mounted = useRef(true);

  useEffect(() => { actionRef.current = action; }, [action]);
  useEffect(() => { mounted.current = true; return () => { mounted.current = false; }; }, []);

  const run = useCallback(async (...args: A): Promise<R | undefined> => {
    if (running.current) return undefined;
    running.current = true;
    setState("busy");
    try {
      const result = await actionRef.current(...args);
      if (mounted.current) {
        setState(doneMs > 0 ? "done" : "idle");
        if (doneMs > 0) window.setTimeout(() => mounted.current && setState("idle"), doneMs);
      }
      return result;
    } catch (error) {
      if (mounted.current) setState("idle");
      throw error;
    } finally {
      running.current = false;
    }
  }, [doneMs]);

  return { run, busy: state === "busy", done: state === "done" };
}
```

## Toast.tsx

Mount `<ToastProvider>` at the root layout so marketing and product pages
share one stack and a toast raised just before a navigation survives it.

```tsx
"use client";

import { createContext, useCallback, useContext, useEffect, useMemo, useRef, useState } from "react";

type Tone = "ok" | "info" | "bad";
type ToastAction = { label: string; run: () => void };
type Toast = { id: number; tone: Tone; message: React.ReactNode; action?: ToastAction; leaving: boolean };
type ToastApi = Record<"success" | "info" | "error", (message: React.ReactNode, options?: { action?: ToastAction }) => void>;

const MAX = 3;        // a fourth pushes out the oldest
const STAY_MS = 4000; // success and info leave on their own; errors stay
const LEAVE_MS = 160;

const noop = () => undefined;
const ToastContext = createContext<ToastApi>({ success: noop, info: noop, error: noop });
export const useToast = () => useContext(ToastContext);

export function ToastProvider({ children }: { children: React.ReactNode }) {
  const [toasts, setToasts] = useState<Toast[]>([]);
  const nextId = useRef(1);

  const dismiss = useCallback((id: number) => {
    setToasts((all) => all.map((t) => (t.id === id ? { ...t, leaving: true } : t)));
    window.setTimeout(() => setToasts((all) => all.filter((t) => t.id !== id)), LEAVE_MS);
  }, []);

  const push = useCallback((tone: Tone, message: React.ReactNode, action?: ToastAction) => {
    const id = nextId.current++;
    setToasts((all) => [...all, { id, tone, message, action, leaving: false }].slice(-MAX));
  }, []);

  const api = useMemo<ToastApi>(() => ({
    success: (m, o) => push("ok", m, o?.action),
    info: (m, o) => push("info", m, o?.action),
    error: (m, o) => push("bad", m, o?.action),
  }), [push]);

  return (
    <ToastContext.Provider value={api}>
      {children}
      <section className="toast-region" aria-label="Notifications" aria-live="polite">
        {toasts.map((t) => <ToastItem key={t.id} toast={t} onDismiss={dismiss} />)}
      </section>
    </ToastContext.Provider>
  );
}

function ToastItem({ toast, onDismiss }: { toast: Toast; onDismiss: (id: number) => void }) {
  const [paused, setPaused] = useState(false);
  const sticky = toast.tone === "bad";

  // A pointer resting on the toast, or focus inside it, holds it on screen.
  useEffect(() => {
    if (sticky || paused || toast.leaving) return;
    const timer = window.setTimeout(() => onDismiss(toast.id), STAY_MS);
    return () => window.clearTimeout(timer);
  }, [sticky, paused, toast.leaving, toast.id, onDismiss]);

  return (
    <div
      className={`toast toast-${toast.tone}${toast.leaving ? " toast-leaving" : ""}`}
      role={sticky ? "alert" : "status"}
      onPointerEnter={() => setPaused(true)}
      onPointerLeave={() => setPaused(false)}
      onFocus={() => setPaused(true)}
      onBlur={() => setPaused(false)}
    >
      <span className="toast-text">
        <span>{toast.message}</span>
        {toast.action && (
          <button type="button" className="toast-action" onClick={() => { onDismiss(toast.id); toast.action?.run(); }}>
            {toast.action.label}
          </button>
        )}
      </span>
      <button type="button" className="toast-close" aria-label="Dismiss" onClick={() => onDismiss(toast.id)}>×</button>
    </div>
  );
}
```

Toast CSS essentials: `.toast-region { position: fixed; right: 24px; bottom: 24px; z-index: 1000; display: flex; flex-direction: column; gap: 10px; width: min(380px, calc(100vw - 32px)); pointer-events: none }`, `.toast { pointer-events: auto; animation: enter var(--t-base) var(--ease-out); transition: opacity 160ms, transform 160ms }`, `.toast-leaving { opacity: 0; transform: translateY(6px) }`. On phones, centre the region and lift it above any bottom tab bar and `env(safe-area-inset-bottom)`.

## Link pending (Next.js 15.3+)

Each component reads the nearest enclosing `<Link>`, so render it INSIDE the
link. Keep `LinkSpinner` in a module with no stylesheet import, so marketing
pages can use it without loading the product's CSS.

```tsx
// LinkSpinner.tsx
"use client";
import { useLinkStatus } from "next/link";
import { Spinner } from "./Button";

/** A button-styled link: the busy spinner while the next page loads. */
export function LinkSpinner() {
  const { pending } = useLinkStatus();
  return pending ? <Spinner /> : null;
}
```

```tsx
// LinkPending.tsx (product only; may import the product stylesheet)
"use client";
import { useLinkStatus } from "next/link";

/** Nav item: a breathing dot at the end of the row. */
export function NavPending() {
  const { pending } = useLinkStatus();
  return pending ? <span className="nav-pending" aria-hidden="true" /> : null;
}

/** Card link: a 2px bar grows across the top edge after 120ms (no flash on fast loads). The card needs position: relative. */
export function CardPending() {
  const { pending } = useLinkStatus();
  return pending ? <span className="card-pending" aria-hidden="true" /> : null;
}
```

```css
.nav-pending { margin-left: auto; width: 6px; height: 6px; border-radius: 999px; background: var(--accent); animation: breathe 1s ease-in-out infinite; }
.card-pending { position: absolute; left: 0; top: 0; z-index: 2; height: 2px; width: 0; background: var(--accent); animation: card-pending 1.6s var(--ease-out) 120ms forwards; }
@keyframes card-pending { to { width: 82%; } }
```

## Page fade (Next.js app router)

`app/template.tsx`, plus one per route group whose shell should stay still
(for example `app/(app)/template.tsx`), renders `<RouteFade>{children}</RouteFade>`.
A template remounts on navigation between its children; that remount is the
cue for the fade.

```tsx
"use client";

import { useEffect, useState } from "react";

/** Whether this tab has painted a page yet: resets on a full reload. */
let painted = false;

/**
 * Fades a screen in on client-side navigation only. The first page load is
 * never faded: opacity 0 is not a paint candidate, so a fade would push back
 * first contentful paint. Server and hydrating renders both see `false`.
 */
export function RouteFade({ className, children }: { className?: string; children: React.ReactNode }) {
  // Decided once, at mount. A template re-renders without remounting when a
  // navigation happens below it; re-reading `painted` then would fade the
  // whole shell instead of only the screen that changed.
  const [fade] = useState(() => painted);
  useEffect(() => { painted = true; }, []);
  return <div className={`${fade ? "route-in" : ""} ${className ?? ""}`.trim() || undefined}>{children}</div>;
}
```

Inside a flex-column shell, pass a class giving the wrapper
`display: flex; flex-direction: column; flex-grow: 1; min-height: 0` so the
screen body still scrolls.

## Skeleton components

One per main screen, server-safe (no hooks, no "use client"), reusing the
real layout classes so the placeholder occupies exactly the space the content
will. Use the same component in the route's `loading.tsx` and in the page's
first-read branch so they hand over without a flash. A `loading.tsx` also
covers nested routes: give deeper routes (`/x/new`, `/x/[id]`) their own.

```tsx
type Box = { w?: number | string; h: number; r?: number | string; style?: React.CSSProperties };

/** One skeleton block; `.sk` owns the colour and the sweep. */
export function Sk({ w = "100%", h, r = 6, style }: Box) {
  return <div className="sk" style={{ width: w, height: h, borderRadius: r, flexShrink: 0, ...style }} />;
}

/** Announced once as loading; the blocks themselves are hidden from assistive tech. */
function Placeholders({ label, children }: { label: string; children: React.ReactNode }) {
  return (
    <div role="status" aria-label={label} style={{ display: "contents" }}>
      <div aria-hidden="true" style={{ display: "contents" }}>{children}</div>
    </div>
  );
}

export function CardGridSkeleton({ count = 8 }: { count?: number }) {
  return (
    <Placeholders label="Loading">
      <div className="grid">{/* the page's real grid class */}
        {Array.from({ length: count }, (_, i) => (
          <div key={i} className="card">
            <Sk h={186} r={0} />
            <div style={{ padding: 14, display: "grid", gap: 10 }}>
              <Sk w="92%" h={13} />
              <Sk w="58%" h={13} />
            </div>
          </div>
        ))}
      </div>
    </Placeholders>
  );
}

// app/items/loading.tsx
// export default function Loading() { return <Screen title="Items"><CardGridSkeleton /></Screen>; }
```

## Optimistic patterns

Flip, then reconcile:

```tsx
const toggle = async () => {
  const previous = item;
  const optimistic = { ...item, status: item.status === "paused" ? "active" : "paused" };
  onChanged(optimistic);                                       // the UI flips now
  try {
    onChanged(await api.setStatus(item.id, optimistic.status)); // server truth wins
  } catch (cause) {
    onChanged(previous);                                       // put it back
    setError(messageFor(cause)); // inline if visible, else toast.error(msg, { action: { label: "Try again", run: toggle } })
  }
};
```

Delete that navigates away before the request lands: keep a module-level set
of ids being removed, which list screens filter out, so a list read taken
before the delete landed cannot show the item again.

```ts
const removing = new Set<string>();
const listeners = new Set<() => void>();
let snapshot: ReadonlySet<string> = new Set();
const emit = () => { snapshot = new Set(removing); listeners.forEach((l) => l()); };

export const addPendingRemoval = (id: string) => { removing.add(id); emit(); };
export const removePendingRemoval = (id: string) => { removing.delete(id); emit(); };
export const subscribePendingRemovals = (l: () => void) => { listeners.add(l); return () => listeners.delete(l); };
export const usePendingRemovals = () =>
  useSyncExternalStore(subscribePendingRemovals, () => snapshot, () => new Set<string>());

export async function deleteBehind(id: string, toast: ToastApi, remove: (id: string) => Promise<void>) {
  addPendingRemoval(id);
  try {
    await remove(id);
    toast.success("Deleted.");
    // The id stays hidden: ids are never reused, so this only hides stale copies.
  } catch {
    removePendingRemoval(id);
    toast.error("Could not delete it. It is back in the list.", {
      action: { label: "Try again", run: () => void deleteBehind(id, toast, remove) },
    });
  }
}
```
