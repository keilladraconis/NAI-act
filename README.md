# NAIAct

**NAIAct** is a small, disciplined runtime for building **interactive NovelAI Script UIs** on top of NovelAI's **retained, imperative UI API**.

NAIAct exists to solve one problem well:

> How do you bind application state to a retained UI _without_ re-rendering structure, leaking lifecycle bugs, or introducing implicit global state?

NAIAct borrows ideas from React and Redux, but it is **not a React clone**. There is no virtual DOM, no render loop, and no reconciliation. Instead, NAIAct uses a build-once model where components create their UI structure and bind dynamic behavior in a single pass.

## Installation

### Method A: Copy-paste (simplest)

Copy `src/nai-act.ts` directly into your NovelAI Script project.

### Method B: npm + nibs

If your project uses [nibs](https://github.com/LaneRendell/NovelAI_Script_BuildSystem) or another bundler that resolves `node_modules`:

```bash
npm install nai-act
```

```ts
import { defineComponent, mount, mergeStyles } from "nai-act";
```

> **Note:** This package distributes raw TypeScript source — no compilation step is needed. Your bundler must support `.ts` imports.

---

## The Core Mental Model

NAIAct enforces a **two-phase model**:

1. **Build** — `mount()` calls the component's `build()` method, which returns static UI structure while simultaneously binding dynamic behavior to state.
2. **Register** — The resulting UI parts are passed to NovelAI's `api.v1.ui.register()`.

- UI structure is created once during build.
- State changes drive targeted updates via element IDs.
- Behavior (subscriptions, effects) is bound during build and cleaned up on unmount.

---

## Components

A **Component** is a small unit that creates UI structure and binds it to state.

```typescript
const MyComponent = defineComponent({
  // Unique ID generator for this component instance
  id(props: { label: string }) {
    return "my-component";
  },

  // Build: create structure and bind behavior
  build(props, ctx) {
    return api.v1.ui.part.text({
      id: this.id(props),
      // bindPart reads initial state and subscribes for updates in one call
      ...ctx.bindPart(this.id(props), (s) => s.label, (label) => ({ text: label })),
    });
  },
});
```

The `build` method serves two purposes:

1. **Returns** the initial UI structure (a `UIPart`).
2. **Binds** dynamic behavior via `ctx` (subscriptions, effects, child renders).

These are conceptually separate — the return value is structure, the side effects are behavior — but they're defined together in one place for clarity and co-location.

---

## Canonical Example: Counter

### 1. The Component

```typescript
const Counter = defineComponent({
  id() {
    return "counter";
  },

  build(_, ctx) {
    return api.v1.ui.part.button({
      id: this.id(),
      // bindPart reads initial state and subscribes for updates — one path, no divergence
      ...ctx.bindPart(this.id(), (s) => s.counter, (count) => ({
        text: `Count: ${count}`,
      })),
      callback: () => ctx.dispatch({ type: "INC" }),
    });
  },
});
```

### 2. Build & Register

```typescript
// Create your store (using NAIStore or compatible)
const store = createStore(reducer);

// Build: mount creates UI parts and binds behavior
const counter = mount(Counter, {}, store);

// Register: pass the built parts to NovelAI
await api.v1.ui.register([
  api.v1.ui.extension.scriptPanel({
    id: "panel",
    name: "Counter",
    content: [counter.part],
  }),
]);
```

---

## Complex Example: Todo List

This example demonstrates dynamic lists, child component lifecycles, and NAIStore integration.

### State Slice (NAIStore)

```typescript
// Define State Type
type Todo = { id: string };
type State = { todos: Todo[] };

// Create Slice
const todosSlice = createSlice({
  name: "todos",
  initialState: [] as Todo[],
  reducers: {
    add: (state) => [...state, { id: api.v1.uuid() }],
    remove: (state, payload: { id: string }) =>
      state.filter((t) => t.id !== payload.id),
  },
});

const reducer = combineReducers({
  todos: todosSlice.reducer,
});
```

### Child Component: `TodoItem`

```typescript
type TodoItemProps = { id: string };

const TodoItem = defineComponent({
  id(props: TodoItemProps) {
    return `todo:${props.id}`;
  },

  build(props, ctx: BindContext<State>) {
    // Bind behavior: dispatch directly in callbacks
    // (no event proxy needed)

    return api.v1.ui.part.row({
      content: [
        api.v1.ui.part.textInput({
          storageKey: this.id(props),
          placeholder: "I want Todo...",
        }),
        api.v1.ui.part.button({
          text: "✕",
          callback: () =>
            ctx.dispatch(todosSlice.actions.remove({ id: props.id })),
        }),
      ],
    });
  },
});
```

### Parent Component: `TodoList`

```typescript
const TodoList = defineComponent({
  id() {
    return "todo-list";
  },

  build(_, ctx: BindContext<State>) {
    return api.v1.ui.part.column({
      id: this.id(undefined),
      content: ctx.bindList(
        this.id(undefined),
        (s) => s.todos,
        (todo) => todo.id,
        (todo) => ({ component: TodoItem, props: { id: todo.id } }),
      ),
    });
  },
});
```

### Add Button

```typescript
const AddTodoButton = defineComponent({
  id() {
    return "add-todo-btn";
  },

  build(_, ctx: BindContext<State>) {
    return api.v1.ui.part.button({
      id: this.id(),
      text: "Add Todo",
      callback: () => ctx.dispatch(todosSlice.actions.add()),
    });
  },
});
```

### Final Assembly

```typescript
const store = createStore(reducer);

// Build: mount creates parts and binds behavior
const todoList = mount(TodoList, {}, store);
const addButton = mount(AddTodoButton, {}, store);

// Register: pass the built parts to NovelAI
await api.v1.ui.register([
  api.v1.ui.extension.scriptPanel({
    id: "panel",
    name: "Todos",
    content: [todoList.part, addButton.part],
  }),
]);
```

---

## Styling & Dynamic Updates

The NovelAI Scripting API uses `api.v1.ui.updateParts` for updates. This method performs a **shallow merge** on the component's properties.

### The Pitfall: Lost Styles

If your component has styles and you update it without re-supplying those styles, **they will be lost**.

```typescript
// Initial build
api.v1.ui.part.box({ id: 'my-box', style: { padding: '10px' } })

// Bad Update: The padding is lost because 'style' is overwritten!
api.v1.ui.updateParts([{ id: 'my-box', style: { backgroundColor: 'red' } }])
```

### Best Practice: The `styles` Property

NAIAct Components support a `styles` property to define your static styles in one place. When you define `styles`, `defineComponent` automatically injects a `this.style()` method that merges styles by key.

```typescript
const TodoItem = defineComponent({
  // ...
  styles: {
    base: { padding: '10px', backgroundColor: 'white' },
    done: { backgroundColor: '#e8f5e9' } // Light green
  },

  build(props, ctx) {
    return api.v1.ui.part.box({
      id: this.id(props),
      content: [],
      // bindPart handles initial style and subscribes for updates in one call
      ...ctx.bindPart(
        this.id(props),
        (s) => s.todos.find(t => t.id === props.id).done,
        (isDone) => ({ style: this.style?.("base", isDone && "done") }),
      ),
    });
  }
});
```

### The `style()` Method

The `style()` method accepts any number of style keys and merges them in order. Falsy values (`undefined`, `false`, `null`) are ignored:

```typescript
// Single style
this.style?.("base")

// Conditional style
this.style?.("base", isActive && "active")

// Multiple conditionals
this.style?.("base", isHovered && "hover", isDisabled && "disabled")
```

### Manual Merging with `mergeStyles`

For cases where you need to merge styles outside of a component (or with inline style objects), you can use `mergeStyles` directly:

```typescript
import { mergeStyles } from "./nai-act";

const combined = mergeStyles(
  { padding: '10px' },
  isActive ? { backgroundColor: 'blue' } : undefined
);
```

---

## API Reference

### `defineComponent(config)`
Creates a component definition.
- `id`: Function to generate unique IDs based on props.
- `styles`: Optional object mapping style names to style objects.
- `style`: Auto-injected method (when `styles` is defined) that merges styles by key. Accepts `(...keys: (keyof styles | undefined | false | null)[]) => Style`.
- `build(props, ctx)`: Creates UI structure and binds dynamic behavior. Returns `UIPart`.

### `mount(Component, Props, Store)`
Mounts a component to a store.
- Calls `build()` to create UI parts and bind behavior.
- Returns `{ part: UIPart, unmount: () => void }`.

### `mergeStyles(...styles)`
Merges multiple style objects into one. Later styles override earlier ones. Falsy values are ignored.

### `BindContext` (Passed to `build`)
- `getState()`: Get current state.
- `dispatch(action)`: Dispatch an action.
- `useSelector(selector, listener, equals?)`: Subscribe to state changes. Auto-cleaned up on unmount. Use this for non-UIPart side effects (e.g. syncing to `api.v1.memory`). Optional `equals` function controls when the listener fires — defaults to `Object.is`. Useful for derived arrays/objects where reference equality would fire on every dispatch.
- `useEffect(predicate, effect)`: Subscribe to actions (side-effects). Auto-cleaned up on unmount.
- `render(Component, Props)`: Render a child component. Returns `{ part: UIPart, unmount: () => void }`. Auto-cleaned up on parent unmount.
- `bindPart(partId, selector, mapper)`: Binds a UIPart property to state. Reads current state to return initial props (spread into the UIPart definition), and subscribes to call `updateParts` on every subsequent change. Eliminates the two-path `getState()` + `useSelector` + `updateParts` pattern. **Note:** the return value IS the initial mapped props — always spread it into the part definition; discarding it leaves initial content unset.
- `bindList(containerId, selector, keyFn, renderItem)`: Manages a dynamic list of child components. Returns the initial `UIPart[]` for use in `build()`. On structural change (keys added, removed, or reordered), fully remounts all child components from current state so every child's `build()` runs with up-to-date props, then updates the container via `updateParts`. Fires only when the key sequence actually changes (element-wise comparison). All children are cleaned up on parent unmount.
