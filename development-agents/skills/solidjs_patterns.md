---
name: SolidJS Reactive Patterns
description: SolidJS component patterns with signals, stores, effects, and reactive state. THIS IS NOT REACT - use SolidJS primitives.
---

# SolidJS Reactive Patterns

## CRITICAL: This is SolidJS, NOT React!

**SolidJS uses different primitives than React. Never use React hooks!**

| SolidJS | React (DON'T USE) |
|---------|-------------------|
| `createSignal()` | `useState()` |
| `createEffect()` | `useEffect()` |
| `createStore()` | `useState()` |
| `createMemo()` | `useMemo()` |
| `<For>` | `array.map()` |
| `<Show>` | `&&` conditional |

## Signal Pattern (Simple State)

```tsx
import { createSignal } from "solid-js";

const Counter: Component = () => {
    // Signal returns [getter, setter]
    const [count, setCount] = createSignal(0);

    // Getter is a function - must call it
    const increment = () => {
        setCount(count() + 1);  // count() to read
    };

    // Can also use callback form
    const decrement = () => {
        setCount(c => c - 1);   // Callback receives current value
    };

    return (
        <div>
            <p>Count: {count()}</p>  {/* Must call count() */}
            <button onClick={increment}>+</button>
            <button onClick={decrement}>-</button>
        </div>
    );
};
```

**Key points:**
- Signals return `[getter, setter]`
- Getters are functions: `count()` not `count`
- Setter can take value or callback
- Signals are reactive - UI updates automatically

## Store Pattern (Complex State)

```tsx
import { createStore } from "solid-js/store";

interface FormState {
    firstName: string;
    lastName: string;
    email: string;
    preferences: {
        theme: "light" | "dark";
        notifications: boolean;
    };
}

const UserForm: Component = () => {
    const [formState, setFormState] = createStore<FormState>({
        firstName: "",
        lastName: "",
        email: "",
        preferences: {
            theme: "light",
            notifications: true,
        },
    });

    // Update top-level field
    const updateFirstName = (value: string) => {
        setFormState("firstName", value);
    };

    // Update nested field
    const updateTheme = (theme: "light" | "dark") => {
        setFormState("preferences", "theme", theme);
    };

    // Update multiple fields
    const updatePreferences = () => {
        setFormState("preferences", {
            theme: "dark",
            notifications: false,
        });
    };

    return (
        <form>
            <input
                value={formState.firstName}
                onInput={(e) => setFormState("firstName", e.currentTarget.value)}
            />
            <input
                value={formState.email}
                onInput={(e) => setFormState("email", e.currentTarget.value)}
            />
        </form>
    );
};
```

**When to use stores:**
- Forms with multiple fields
- Nested state objects
- Arrays that need reactivity
- State that needs granular updates

## Memo Pattern (Computed Values)

```tsx
import { createSignal, createMemo } from "solid-js";

const UserProfile: Component = () => {
    const [firstName, setFirstName] = createSignal("John");
    const [lastName, setLastName] = createSignal("Doe");

    // Memoized computed value
    const fullName = createMemo(() => {
        return `${firstName()} ${lastName()}`;
    });

    // Only recalculates when firstName or lastName changes
    const initials = createMemo(() => {
        return `${firstName()[0]}${lastName()[0]}`;
    });

    return (
        <div>
            <p>Full Name: {fullName()}</p>
            <p>Initials: {initials()}</p>
        </div>
    );
};
```

**Use memos for:**
- Expensive computations
- Derived state
- Filtering/mapping data
- Values used in multiple places

## Effect Pattern (Side Effects)

```tsx
import { createSignal, createEffect } from "solid-js";

const DataFetcher: Component = () => {
    const [userId, setUserId] = createSignal("123");
    const [userData, setUserData] = createSignal(null);

    // Effect runs when dependencies change
    createEffect(() => {
        const id = userId();  // Dependency tracking

        fetch(`/api/users/${id}`)
            .then(res => res.json())
            .then(data => setUserData(data));
    });

    return <div>{/* ... */}</div>;
};
```

**Use effects for:**
- API calls
- Local storage updates
- DOM manipulation
- Logging/analytics
- Subscriptions

## Component Pattern

```tsx
import { Component, createSignal } from "solid-js";
import styles from "./MyComponent.module.scss";

interface MyComponentProps {
    title: string;
    initialCount?: number;
    onSubmit?: (value: number) => void;
}

const MyComponent: Component<MyComponentProps> = (props) => {
    const [count, setCount] = createSignal(props.initialCount || 0);

    const handleSubmit = () => {
        props.onSubmit?.(count());
    };

    return (
        <div class={styles.container}>
            <h2>{props.title}</h2>
            <p>Count: {count()}</p>
            <button onClick={() => setCount(c => c + 1)}>Increment</button>
            <button onClick={handleSubmit}>Submit</button>
        </div>
    );
};

export default MyComponent;
```

**Conventions:**
- Use `Component<Props>` type
- Props interface above component
- CSS modules for styling
- `class` not `className`
- Export default at end

## Conditional Rendering

```tsx
import { Show } from "solid-js";

const ConditionalExample: Component = () => {
    const [user, setUser] = createSignal<User | null>(null);
    const [isLoading, setIsLoading] = createSignal(true);

    return (
        <div>
            {/* Show/hide pattern */}
            <Show when={isLoading()}>
                <Spinner />
            </Show>

            {/* Show with fallback */}
            <Show when={user()} fallback={<p>No user found</p>}>
                {(u) => <UserProfile user={u()} />}
            </Show>

            {/* Multiple conditions */}
            <Show
                when={!isLoading() && user()}
                fallback={<p>Loading or no user...</p>}
            >
                {(u) => <p>Welcome, {u().name}!</p>}
            </Show>
        </div>
    );
};
```

**Use `<Show>` instead of `&&` operator:**
- ✅ `<Show when={condition}><Component /></Show>`
- ❌ `{condition && <Component />}`

## List Rendering

```tsx
import { For, Index } from "solid-js";

interface User {
    id: string;
    name: string;
}

const UserList: Component = () => {
    const [users, setUsers] = createSignal<User[]>([
        { id: "1", name: "Alice" },
        { id: "2", name: "Bob" },
    ]);

    return (
        <div>
            {/* For - use when items have stable identity */}
            <For each={users()}>
                {(user) => (
                    <div class={styles.userCard}>
                        <p>{user.name}</p>
                    </div>
                )}
            </For>

            {/* Index - use when items don't have stable identity */}
            <Index each={users()}>
                {(user, index) => (
                    <div>
                        {index}: {user().name}
                    </div>
                )}
            </Index>
        </div>
    );
};
```

**Use `<For>` instead of `array.map()`:**
- ✅ `<For each={items()}>{(item) => <div>{item.name}</div>}</For>`
- ❌ `{items().map(item => <div>{item.name}</div>)}`

## Event Handling

```tsx
const EventExample: Component = () => {
    const [value, setValue] = createSignal("");

    // Input event (onInput, not onChange)
    const handleInput = (e: InputEvent) => {
        const target = e.currentTarget as HTMLInputElement;
        setValue(target.value);
    };

    // Click event
    const handleClick = (e: MouseEvent) => {
        e.preventDefault();
        console.log("Clicked!");
    };

    // Submit event
    const handleSubmit = (e: SubmitEvent) => {
        e.preventDefault();
        console.log("Submitted:", value());
    };

    return (
        <form onSubmit={handleSubmit}>
            <input
                type="text"
                value={value()}
                onInput={handleInput}
            />
            <button onClick={handleClick}>Click</button>
        </form>
    );
};
```

**Event naming:**
- `onInput` for text inputs (not `onChange`)
- `onClick` for clicks
- `onSubmit` for forms
- camelCase event names

## Refs

```tsx
import { onMount } from "solid-js";

const RefExample: Component = () => {
    let inputRef: HTMLInputElement | undefined;

    onMount(() => {
        // Access ref after mount
        inputRef?.focus();
    });

    return (
        <input
            ref={inputRef}
            type="text"
        />
    );
};
```

**Use `ref` prop, not `useRef()`:**
- ✅ `let inputRef; <input ref={inputRef} />`
- ❌ `const inputRef = useRef()`

## Lifecycle

```tsx
import { onMount, onCleanup, createEffect } from "solid-js";

const LifecycleExample: Component = () => {
    // Runs once after mount
    onMount(() => {
        console.log("Component mounted");
        fetchData();
    });

    // Cleanup function
    onCleanup(() => {
        console.log("Component will unmount");
        cancelSubscription();
    });

    // Effect with cleanup
    createEffect(() => {
        const subscription = subscribe();

        onCleanup(() => {
            subscription.unsubscribe();
        });
    });

    return <div>...</div>;
};
```

## Dynamic Classes

```tsx
import { createSignal } from "solid-js";
import styles from "./Button.module.scss";

const Button: Component<{ variant?: "primary" | "secondary" }> = (props) => {
    const [isActive, setIsActive] = createSignal(false);

    return (
        <button
            class={styles.button}
            classList={{
                [styles.primary]: props.variant === "primary",
                [styles.secondary]: props.variant === "secondary",
                [styles.active]: isActive(),
            }}
            onClick={() => setIsActive(!isActive())}
        >
            Click me
        </button>
    );
};
```

**Use `classList` for conditional classes:**
- ✅ `classList={{ [styles.active]: isActive() }}`
- ❌ `className={isActive() ? styles.active : ""}`

## Context Pattern

```tsx
import { createContext, useContext, JSX } from "solid-js";
import { createStore } from "solid-js/store";

// Context type
type AppContextValue = {
    user: () => User | null;
    theme: () => "light" | "dark";
    setTheme: (theme: "light" | "dark") => void;
};

// Create context
const AppContext = createContext<AppContextValue>();

// Provider component
export function AppProvider(props: { children: JSX.Element }) {
    const [state, setState] = createStore({
        user: null as User | null,
        theme: "light" as "light" | "dark",
    });

    const value: AppContextValue = {
        user: () => state.user,
        theme: () => state.theme,
        setTheme: (theme) => setState("theme", theme),
    };

    return (
        <AppContext.Provider value={value}>
            {props.children}
        </AppContext.Provider>
    );
}

// Hook to use context
export function useApp() {
    const context = useContext(AppContext);
    if (!context) {
        throw new Error("useApp must be used within AppProvider");
    }
    return context;
}
```

## Error Boundary

```tsx
import { ErrorBoundary } from "solid-js";

const App: Component = () => {
    return (
        <ErrorBoundary fallback={(err) => <ErrorView error={err} />}>
            <MainApp />
        </ErrorBoundary>
    );
};

const ErrorView: Component<{ error: Error }> = (props) => {
    return (
        <div>
            <h1>Something went wrong</h1>
            <pre>{props.error.message}</pre>
        </div>
    );
};
```

## Common Anti-Patterns

### ❌ Using React Hooks

```tsx
// DON'T DO THIS
const [state, setState] = useState(0);  // Wrong!
const value = useMemo(() => compute());  // Wrong!
useEffect(() => { /* ... */ });          // Wrong!
```

### ❌ Forgetting to Call Signals

```tsx
// DON'T DO THIS
<p>Count: {count}</p>  // Wrong! count is a function

// DO THIS
<p>Count: {count()}</p>  // Correct!
```

### ❌ Using Array.map()

```tsx
// DON'T DO THIS
{items().map(item => <div>{item.name}</div>)}

// DO THIS
<For each={items()}>
    {(item) => <div>{item.name}</div>}
</For>
```

### ❌ Using && for Conditionals

```tsx
// DON'T DO THIS
{condition && <Component />}

// DO THIS
<Show when={condition}>
    <Component />
</Show>
```

## Summary

**Key Patterns:**
1. Use `createSignal()` for simple state
2. Use `createStore()` for complex/nested state
3. Use `createMemo()` for computed values
4. Use `createEffect()` for side effects
5. Use `<For>` for lists, not `array.map()`
6. Use `<Show>` for conditionals, not `&&`
7. Call signals as functions: `count()`
8. Use `Component<Props>` type
9. Use `class` not `className`
10. Use `onInput` not `onChange`

**Remember: This is SolidJS, NOT React!**
