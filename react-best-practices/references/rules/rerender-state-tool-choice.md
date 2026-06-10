---
title: Choose the Right State Tool
impact: MEDIUM
impactDescription: eliminates redundant re-renders and stale UI bugs
tags: rerender, useState, useReducer, useRef, derived-state, state-management
---

## Choose the Right State Tool

Match each value to the primitive that fits how it is used. Storing a value in the wrong place causes redundant re-renders (handler-only data in state) or stale UI (render-affecting data in refs).

- `useState` — values that affect rendered output.
- `useReducer` — render-affecting state with complex transitions (multiple fields updated together, next state depends on previous).
- Derive during render — values computable from existing props/state. Don't store them; compute inline, or `useMemo` if expensive.
- `useEffect` or event handlers — synchronization with external systems only, never transforming state into other state.
- `useRef` — legitimate escape-hatch cases only: DOM access, timer IDs, imperative external objects (map/chart instances), or values needed by handlers but not rendering.

**Incorrect (derivable value stored as state, synced via effect):**

```tsx
function Cart({ items }: { items: Item[] }) {
  const [total, setTotal] = useState(0)

  useEffect(() => {
    setTotal(items.reduce((sum, item) => sum + item.price, 0))
  }, [items])

  return <p>Total: {total}</p>  // renders twice per items change
}
```

**Correct (derive during render):**

```tsx
function Cart({ items }: { items: Item[] }) {
  const total = items.reduce((sum, item) => sum + item.price, 0)
  return <p>Total: {total}</p>
}
```

**Incorrect (handler-only value in state — re-renders on every pointer move):**

```tsx
function Canvas() {
  const [lastPos, setLastPos] = useState({ x: 0, y: 0 })

  const handleMove = (e: React.PointerEvent) => {
    drawLine(lastPos, { x: e.clientX, y: e.clientY })
    setLastPos({ x: e.clientX, y: e.clientY })
  }

  return <canvas onPointerMove={handleMove} />
}
```

**Correct (handler-only value in a ref — no re-renders):**

```tsx
function Canvas() {
  const lastPos = useRef({ x: 0, y: 0 })

  const handleMove = (e: React.PointerEvent) => {
    drawLine(lastPos.current, { x: e.clientX, y: e.clientY })
    lastPos.current = { x: e.clientX, y: e.clientY }
  }

  return <canvas onPointerMove={handleMove} />
}
```

For subscription granularity of derived values, see "Subscribe to Derived State". For refs holding callbacks used in effects, see "Store Event Handlers in Refs" and "useLatest for Stable Callback Refs".
