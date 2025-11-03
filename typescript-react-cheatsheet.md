# TypeScript + React Cheat Sheet (Markdown)

A compact, practical reference for building React apps with TypeScript. Copy-paste friendly, with the most useful patterns and types you'll hit every day.

---

## Project Setup (quick)

```bash
# Vite (recommended)
npm create vite@latest my-app -- --template react-ts
cd my-app && npm i

# React types (already included with react-ts)
npm i -D typescript @types/react @types/react-dom
```

`tsconfig.json` essentials:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["DOM", "ES2022"],
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "moduleResolution": "Bundler",
    "module": "ESNext",
    "skipLibCheck": true,
    "esModuleInterop": true
  }
}
```

---

## React Types You’ll Use Constantly

```ts
import type {
  ReactNode, ReactElement, ReactPortal,
  FC, PropsWithChildren,
  ComponentProps, ComponentPropsWithoutRef, ComponentPropsWithRef,
  JSX
} from "react";
```

* `ReactNode` — anything renderable (`string | number | JSX | null | undefined`…).
* `ReactElement` — a concrete JSX element returned from a component.
* `PropsWithChildren<P>` — add `children?: ReactNode` to your props.
* `ComponentProps<"button">` — props of an intrinsic element (HTML tag).
* `ComponentProps<typeof MyComp>` — props of another component.
* Prefer **not** to use `FC` unless you need its `children` & default `props` helpers.

---

## Props & State

### Function Components (preferred)

```tsx
type ButtonProps = {
  kind?: "primary" | "ghost";
  onClick?: () => void;
  disabled?: boolean;
  children: React.ReactNode;
};

export function Button({ kind = "primary", onClick, disabled, children }: ButtonProps) {
  return (
    <button data-kind={kind} onClick={onClick} disabled={disabled}>
      {children}
    </button>
  );
}
```

### `useState` Typing

```tsx
const [count, setCount] = React.useState(0);                    // number inferred
const [name, setName] = React.useState<string>("");             // explicit
const [user, setUser] = React.useState<User | null>(null);      // unions
setCount(c => c + 1);                                           // functional update
```

Lazy init:

```tsx
const [map] = React.useState(() => new Map<string, number>());
```

### Derived Props & Defaults

```tsx
type CardProps = {
  title: string;
  subtitle?: string;
  footer?: React.ReactNode;
} & React.ComponentPropsWithoutRef<"section">;

function Card({ title, subtitle = "—", footer, ...rest }: CardProps) {
  return (
    <section {...rest}>
      <h3>{title}</h3>
      <p>{subtitle}</p>
      {footer}
    </section>
  );
}
```

---

## Children Patterns

```tsx
// PropsWithChildren
type PanelProps = React.PropsWithChildren<{ heading: string }>;
function Panel({ heading, children }: PanelProps) {
  return (<div><h2>{heading}</h2>{children}</div>);
}

// Render props
type ListProps<T> = { items: T[]; renderItem: (item: T) => React.ReactNode };
function List<T>({ items, renderItem }: ListProps<T>) {
  return <ul>{items.map(renderItem)}</ul>;
}
```

---

## Events & DOM Types

Common event types:

```ts
import type {
  ChangeEvent, MouseEvent, KeyboardEvent, FormEvent, FocusEvent
} from "react";
```

Usage:

```tsx
function Input() {
  const onChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    console.log(e.currentTarget.value);
  };
  return <input onChange={onChange} />;
}

function LinkLike() {
  const onClick = (e: React.MouseEvent<HTMLButtonElement>) => {
    e.preventDefault();
  };
  return <button onClick={onClick}>Click</button>;
}
```

Other handy types:

* `React.FormEvent<HTMLFormElement>`
* `React.KeyboardEvent<HTMLInputElement>`
* `React.FocusEvent<HTMLInputElement>`
* `React.DragEvent<HTMLDivElement>`

HTML attributes:

```ts
type ButtonHTMLProps = React.ComponentPropsWithoutRef<"button">;
type InputHTMLProps = React.InputHTMLAttributes<HTMLInputElement>;
```

Inline style:

```tsx
const style: React.CSSProperties = { display: "grid", gap: 8 };
```

---

## Refs

```tsx
function Focusable() {
  const inputRef = React.useRef<HTMLInputElement>(null);

  React.useEffect(() => inputRef.current?.focus(), []);

  return <input ref={inputRef} />;
}
```

### `forwardRef` + `useImperativeHandle`

```tsx
type FancyInputHandle = { focus: () => void };

const FancyInput = React.forwardRef<FancyInputHandle, { label: string }>(
  ({ label }, ref) => {
    const inner = React.useRef<HTMLInputElement>(null);
    React.useImperativeHandle(ref, () => ({
      focus: () => inner.current?.focus(),
    }));
    return <label>{label}<input ref={inner} /></label>;
  }
);

function Parent() {
  const api = React.useRef<FancyInputHandle>(null);
  return (
    <>
      <FancyInput ref={api} label="Name" />
      <button onClick={() => api.current?.focus()}>Focus</button>
    </>
  );
}
```

---

## Reducers

```tsx
type State = { count: number };
type Action =
  | { type: "inc"; by?: number }
  | { type: "dec"; by?: number }
  | { type: "reset" };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "inc": return { count: state.count + (action.by ?? 1) };
    case "dec": return { count: state.count - (action.by ?? 1) };
    case "reset": return { count: 0 };
  }
}

const [state, dispatch] = React.useReducer(reducer, { count: 0 });
```

---

## Context (type-safe)

```tsx
type Auth = { user: { id: string; name: string } | null };

const AuthContext = React.createContext<Auth | undefined>(undefined);

export function AuthProvider({ children }: React.PropsWithChildren<{}>) {
  const [auth, setAuth] = React.useState<Auth>({ user: null });
  const value = React.useMemo(() => auth, [auth]);
  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const ctx = React.useContext(AuthContext);
  if (!ctx) throw new Error("useAuth must be inside <AuthProvider>");
  return ctx;
}
```

---

## Custom Hooks

```ts
function useBoolean(initial = false) {
  const [value, set] = React.useState(initial);
  const on = React.useCallback(() => set(true), []);
  const off = React.useCallback(() => set(false), []);
  const toggle = React.useCallback(() => set(v => !v), []);
  return { value, on, off, toggle } as const; // readonly tuple-like
}
```

---

## Memoization & Callbacks

```tsx
const expensive = React.useMemo(() => compute(data), [data]);
const onSelect = React.useCallback((id: string) => {
  // ...
}, []);
export default React.memo(MyComponent);
```

Type a callback:

```ts
type OnSelect = (id: string) => void;
const onSelect: OnSelect = (id) => {/*...*/};
```

---

## JSX & Intrinsic Elements

Typing a component that renders as different tags:

```tsx
type AsProp<E extends React.ElementType> = {
  as?: E;
} & React.ComponentPropsWithoutRef<E>;

function Box<E extends React.ElementType = "div">({ as, ...rest }: AsProp<E>) {
  const Comp = as ?? "div";
  return <Comp {...rest} />;
}

<Box as="button" onClick={() => {}}>Click</Box>;
```

---

## Discriminated Unions for UI State

```ts
type LoadState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: string };

function view<T>(state: LoadState<T>) {
  switch (state.status) {
    case "idle": return "Start";
    case "loading": return "Loading…";
    case "success": return JSON.stringify(state.data);
    case "error": return state.error;
  }
}
```

---

## Fetch & API Types

### `fetch` with typed result

```ts
type User = { id: number; name: string };

async function getUsers(): Promise<User[]> {
  const res = await fetch("/api/users");
  if (!res.ok) throw new Error("Network error");
  return (await res.json()) as User[];
}
```

### Axios generics (if you use Axios)

```ts
import axios, { AxiosResponse } from "axios";

type Post = { id: number; title: string };

async function getPosts() {
  const res: AxiosResponse<Post[]> = await axios.get("/api/posts");
  return res.data;
}
```

---

## Forms

```tsx
function Form() {
  const [form, setForm] = React.useState({ email: "", age: 0 });

  const onChange = <K extends keyof typeof form>(key: K) =>
    (e: React.ChangeEvent<HTMLInputElement>) => {
      const value = e.currentTarget.type === "number"
        ? Number(e.currentTarget.value) : e.currentTarget.value;
      setForm(f => ({ ...f, [key]: value }));
    };

  const onSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    // form is fully typed
  };

  return (
    <form onSubmit={onSubmit}>
      <input type="email" value={form.email} onChange={onChange("email")} />
      <input type="number" value={form.age} onChange={onChange("age")} />
      <button>Save</button>
    </form>
  );
}
```

---

## Error Boundaries (class component types)

```tsx
type EBState = { hasError: boolean };

export class ErrorBoundary extends React.Component<React.PropsWithChildren, EBState> {
  state: EBState = { hasError: false };

  static getDerivedStateFromError() { return { hasError: true }; }
  componentDidCatch(error: unknown, info: React.ErrorInfo) {
    console.error(error, info);
  }

  render() {
    if (this.state.hasError) return <div>Something went wrong.</div>;
    return this.props.children;
  }
}
```

---

## Suspense & `lazy`

```tsx
const Page = React.lazy(() => import("./Page"));
export default function App() {
  return (
    <React.Suspense fallback={<div>Loading…</div>}>
      <Page />
    </React.Suspense>
  );
}
```

---

## Higher-Order Components (typing)

```tsx
function withLogger<P>(Comp: React.ComponentType<P>) {
  return function Logged(props: P) {
    console.log("render", Comp.name, props);
    return <Comp {...props} />;
  };
}

// Usage
type HelloProps = { name: string };
function Hello({ name }: HelloProps) { return <div>Hello {name}</div>; }
const LoggedHello = withLogger(Hello);
```

---

## CSS Modules & Styled Systems

```tsx
import styles from "./Box.module.css"; // .module.css

type BoxProps = { className?: string } & React.ComponentPropsWithoutRef<"div">;

function Box({ className, ...rest }: BoxProps) {
  return <div className={[styles.box, className].filter(Boolean).join(" ")} {...rest} />;
}
```

---

## Useful Utility Types (built-ins)

```ts
type T0 = Partial<User>;        // all props optional
type T1 = Required<User>;       // all props required
type T2 = Readonly<User>;       // all props readonly
type T3 = Pick<User, "id"|"name">;
type T4 = Omit<User, "password">;
type T5 = Record<string, number>;
type T6 = NonNullable<string | null | undefined>;
type T7 = ReturnType<typeof myFn>;
type T8 = Parameters<typeof myFn>;  // tuple of args
type T9 = Extract<"a"|"b", "b"|"c">; // "b"
type T10 = Exclude<"a"|"b","b">; // "a"
```

---

## Type Narrowing Patterns

```ts
function isDefined<T>(x: T | null | undefined): x is T {
  return x != null;
}

const xs = [1, null, 2, undefined].filter(isDefined); // number[]
```

Discriminators:

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rect"; w: number; h: number };

function area(s: Shape) {
  if (s.kind === "circle") return Math.PI * s.radius ** 2;
  return s.w * s.h;
}
```

---

## Module Boundaries & `index.ts`

```ts
// src/api/index.ts
export type { User } from "./types";
export { getUsers } from "./users";
```

Consumers now import from a single path.

---

## Testing (Jest/RTL typing nibbles)

```ts
import { render, screen } from "@testing-library/react";
import "@testing-library/jest-dom";

test("renders title", () => {
  render(<Card title="X" />);
  expect(screen.getByText("X")).toBeInTheDocument();
});
```

`tsconfig` test hint (if needed):

```json
{
  "compilerOptions": {
    "types": ["vitest/globals", "@testing-library/jest-dom"]
  }
}
```

---

## Common Gotchas

* **Avoid `any`**. Prefer `unknown` and narrow.
* **`FC` is optional**: using `FC` can obscure default props & `displayName` handling; prefer plain functions with explicit `children?`.
* **Refs**: `useRef<T | null>(null)`; never `useRef<T>(null!)` unless you really must.
* **Event targets**: use `e.currentTarget` (typed), not `e.target`.
* **Context default**: use `Context<T | undefined>` + custom hook to enforce provider presence.
* **Union state**: prefer discriminated unions over boolean flags.

---

## Quick Reference: Event Type Map

| DOM                    | Type                                                                       |
| ---------------------- | -------------------------------------------------------------------------- |
| `onChange` (input)     | `React.ChangeEvent<HTMLInputElement>`                                      |
| `onSubmit` (form)      | `React.FormEvent<HTMLFormElement>`                                         |
| `onClick` (button/div) | `React.MouseEvent<HTMLButtonElement>` / `React.MouseEvent<HTMLDivElement>` |
| `onKeyDown`            | `React.KeyboardEvent<HTMLElement>`                                         |
| `onFocus`/`onBlur`     | `React.FocusEvent<HTMLElement>`                                            |
| `onDragStart`          | `React.DragEvent<HTMLElement>`                                             |
| `onScroll`             | `React.UIEvent<HTMLElement>`                                               |

---

## Example: Typed Data Grid Row & Cell Renderer

```ts
type Trade = {
  id: string;
  symbol: string;
  side: "BUY" | "SELL";
  qty: number;
  price: number;
  ts: number;
};

type CellRendererProps = {
  value: number;
  row: Trade;
};

function PnLCell({ value, row }: CellRendererProps) {
  const sign = value >= 0 ? "+" : "-";
  return <span aria-label={`${row.symbol}-pnl`}>{sign}{Math.abs(value).toFixed(2)}</span>;
}
```

---

## Example: Strongly-Typed Fetch Hook

```ts
function useFetch<T>(url: string) {
  const [state, set] = React.useState<LoadState<T>>({ status: "idle" });

  React.useEffect(() => {
    let alive = true;
    set({ status: "loading" });
    fetch(url)
      .then(r => r.ok ? r.json() : Promise.reject(r.statusText))
      .then((data: T) => alive && set({ status: "success", data }))
      .catch((e: unknown) => alive && set({ status: "error", error: String(e) }));
    return () => { alive = false; };
  }, [url]);

  return state;
}
```

---

## Example: Polymorphic `as` Component with Ref

```ts
type PolymorphicProps<E extends React.ElementType, P> =
  P & Omit<React.ComponentPropsWithRef<E>, keyof P | "as"> & { as?: E };

type PolymorphicRef<E extends React.ElementType> = React.ComponentPropsWithRef<E>["ref"];

type TextProps = { weight?: "normal" | "bold" };

const Text = React.forwardRef(
  <E extends React.ElementType = "span">(
    { as, weight = "normal", style, ...rest }: PolymorphicProps<E, TextProps>,
    ref: PolymorphicRef<E>
  ) => {
    const Comp = as ?? "span";
    return <Comp ref={ref} style={{ fontWeight: weight, ...style }} {...rest} />;
  }
);
```

---

## Minimal ESLint + TS Rules (optional but helpful)

* `eslint-config-react-app` or `eslint-config-next` (project dependent)
* Key rules to consider:

  * `@typescript-eslint/explicit-function-return-type` (off for components)
  * `@typescript-eslint/no-unused-vars`
  * `@typescript-eslint/consistent-type-imports`
  * `react-hooks/rules-of-hooks` & `react-hooks/exhaustive-deps`

---

## TL;DR Templates

**Component with props + children**

```tsx
type Props = React.PropsWithChildren<{ title: string }>;
export function Section({ title, children }: Props) {
  return (<section><h2>{title}</h2>{children}</section>);
}
```

**Ref + forwardRef**

```tsx
const Input = React.forwardRef<HTMLInputElement, React.ComponentPropsWithoutRef<"input">>(
  (props, ref) => <input ref={ref} {...props} />
);
```

**Reducer with discriminated actions**

```ts
type A = { type: "open" } | { type: "close" } | { type: "set"; value: number };
```

**Typed list `renderItem`**

```tsx
function List<T>({ items, renderItem }: { items: T[]; renderItem: (t: T) => React.ReactNode }) {
  return <ul>{items.map(renderItem)}</ul>;
}
```
