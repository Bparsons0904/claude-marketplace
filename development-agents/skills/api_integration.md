---
name: API Integration Patterns
description: TanStack Solid Query integration with custom hooks, error handling, retry logic, optimistic updates, and cache management.
---

# API Integration Patterns

## TanStack Solid Query (NOT React Query!)

**Important: Use `@tanstack/solid-query`, NOT `@tanstack/react-query`**

## Setup

```tsx
// main.tsx
import { QueryClient, QueryClientProvider } from "@tanstack/solid-query";

const queryClient = new QueryClient({
    defaultOptions: {
        queries: {
            refetchOnWindowFocus: false,
            retry: 3,
            staleTime: 5 * 60 * 1000, // 5 minutes
        },
    },
});

render(() => (
    <QueryClientProvider client={queryClient}>
        <App />
    </QueryClientProvider>
), document.getElementById("root")!);
```

## Generic Query Hook

```tsx
// services/apiHooks.ts
import { useQuery, type UseQueryResult } from "@tanstack/solid-query";
import { api } from "./api";

export function useApiQuery<T>(
    queryKey: readonly unknown[],
    url: string,
    options?: {
        enabled?: boolean | (() => boolean);
        staleTime?: number;
    }
): UseQueryResult<T, Error> {
    const { enabled, ...restOptions } = options || {};

    return useQuery(() => ({
        queryKey,
        queryFn: () => api.get<T>(url),
        refetchOnWindowFocus: false,
        staleTime: 5 * 60 * 1000,
        enabled: typeof enabled === "function" ? enabled() : enabled,
        ...restOptions,
    }));
}
```

## Using Query Hooks

```tsx
import { useApiQuery } from "@services/apiHooks";
import { USER_ENDPOINTS } from "@constants/api.constants";
import type { User } from "@types/User";

const UserProfile: Component = () => {
    const userQuery = useApiQuery<User>(
        ["user"],
        USER_ENDPOINTS.ME
    );

    return (
        <div>
            {/* Loading state */}
            <Show when={userQuery.isPending}>
                <Spinner />
            </Show>

            {/* Error state */}
            <Show when={userQuery.error}>
                <ErrorMessage message={userQuery.error?.message} />
            </Show>

            {/* Success state */}
            <Show when={userQuery.data}>
                {(user) => (
                    <div>
                        <h1>{user().displayName}</h1>
                        <p>{user().email}</p>
                    </div>
                )}
            </Show>
        </div>
    );
};
```

## Generic Mutation Hook

```tsx
import { useMutation, useQueryClient, type UseMutationResult } from "@tanstack/solid-query";
import { api } from "./api";
import { useToast } from "@context/ToastContext";

export function useApiPost<T, V = unknown>(
    url: string | ((variables: V) => string),
    options?: {
        invalidateQueries?: readonly (readonly unknown[])[];
        successMessage?: string | ((data: T, variables: V) => string);
        errorMessage?: string | ((error: Error) => string);
        onSuccess?: (data: T, variables: V) => void;
    }
): UseMutationResult<T, Error, V> {
    const queryClient = useQueryClient();
    const toast = useToast();

    return useMutation(() => ({
        mutationFn: (variables: V) => {
            const requestUrl = typeof url === "function" ? url(variables) : url;
            return api.post<T>(requestUrl, variables);
        },
        onSuccess: (data, variables) => {
            // Invalidate queries
            if (options?.invalidateQueries) {
                options.invalidateQueries.forEach((queryKey) => {
                    queryClient.invalidateQueries({ queryKey });
                });
            }

            // Show success message
            if (options?.successMessage) {
                const message = typeof options.successMessage === "function"
                    ? options.successMessage(data, variables)
                    : options.successMessage;
                toast.success(message);
            }

            // Call user callback
            options?.onSuccess?.(data, variables);
        },
        onError: (error: Error) => {
            // Show error message
            const message = options?.errorMessage
                ? typeof options.errorMessage === "function"
                    ? options.errorMessage(error)
                    : options.errorMessage
                : error.message;
            toast.error(message);
        },
    }));
}
```

## Using Mutation Hooks

```tsx
import { useApiPost } from "@services/apiHooks";
import { USER_ENDPOINTS } from "@constants/api.constants";
import type { User, CreateUserRequest } from "@types/User";

const CreateUserForm: Component = () => {
    const [formState, setFormState] = createStore<CreateUserRequest>({
        firstName: "",
        lastName: "",
        email: "",
    });

    const createUserMutation = useApiPost<User, CreateUserRequest>(
        USER_ENDPOINTS.CREATE,
        {
            invalidateQueries: [["users"]],
            successMessage: "User created successfully!",
            errorMessage: "Failed to create user",
            onSuccess: (user) => {
                console.log("Created user:", user);
                // Reset form or redirect
            },
        }
    );

    const handleSubmit = (e: Event) => {
        e.preventDefault();
        createUserMutation.mutate(formState);
    };

    return (
        <form onSubmit={handleSubmit}>
            <TextInput
                name="firstName"
                label="First Name"
                value={formState.firstName}
                onInput={(value) => setFormState("firstName", value)}
            />
            <button
                type="submit"
                disabled={createUserMutation.isPending}
            >
                {createUserMutation.isPending ? "Creating..." : "Create User"}
            </button>
        </form>
    );
};
```

## Optimistic Updates

```tsx
import { useQueryClient } from "@tanstack/solid-query";

const useUpdateUser = () => {
    const queryClient = useQueryClient();

    return useApiPut<User, UpdateUserRequest>(
        (variables) => `/api/users/${variables.id}`,
        {
            onMutate: async (variables) => {
                // Cancel outgoing refetches
                await queryClient.cancelQueries({ queryKey: ["user", variables.id] });

                // Snapshot previous value
                const previousUser = queryClient.getQueryData<User>(["user", variables.id]);

                // Optimistically update
                queryClient.setQueryData<User>(
                    ["user", variables.id],
                    (old) => {
                        if (!old) return old;
                        return {
                            ...old,
                            ...variables,
                        };
                    }
                );

                // Return context with snapshot
                return { previousUser };
            },
            onError: (error, variables, context) => {
                // Rollback on error
                if (context?.previousUser) {
                    queryClient.setQueryData(
                        ["user", variables.id],
                        context.previousUser
                    );
                }
            },
            onSettled: (data, error, variables) => {
                // Refetch to ensure server state
                queryClient.invalidateQueries({ queryKey: ["user", variables.id] });
            },
        }
    );
};
```

## Query Invalidation

```tsx
import { useQueryClient } from "@tanstack/solid-query";

const UserActions: Component = () => {
    const queryClient = useQueryClient();

    const handleRefresh = () => {
        // Invalidate specific query
        queryClient.invalidateQueries({ queryKey: ["user"] });
    };

    const handleRefreshAll = () => {
        // Invalidate all queries starting with "users"
        queryClient.invalidateQueries({ queryKey: ["users"] });
    };

    const handleClearCache = () => {
        // Remove query from cache
        queryClient.removeQueries({ queryKey: ["user"] });
    };

    return (
        <div>
            <button onClick={handleRefresh}>Refresh User</button>
            <button onClick={handleRefreshAll}>Refresh All Users</button>
            <button onClick={handleClearCache}>Clear Cache</button>
        </div>
    );
};
```

## Dependent Queries

```tsx
const UserWithReleases: Component<{ userId: string }> = (props) => {
    // First query - get user
    const userQuery = useApiQuery<User>(
        ["user", props.userId],
        `/api/users/${props.userId}`
    );

    // Second query - depends on first
    const releasesQuery = useApiQuery<Release[]>(
        ["releases", props.userId],
        `/api/users/${props.userId}/releases`,
        {
            // Only run when user is loaded
            enabled: () => !!userQuery.data,
        }
    );

    return (
        <div>
            <Show when={userQuery.data}>
                {(user) => <UserCard user={user()} />}
            </Show>

            <Show when={releasesQuery.data}>
                {(releases) => (
                    <For each={releases()}>
                        {(release) => <ReleaseCard release={release} />}
                    </For>
                )}
            </Show>
        </div>
    );
};
```

## Paginated Queries

```tsx
const UserList: Component = () => {
    const [page, setPage] = createSignal(1);
    const [limit] = createSignal(20);

    const usersQuery = useApiQuery<{ users: User[]; total: number }>(
        ["users", page(), limit()],
        `/api/users?page=${page()}&limit=${limit()}`
    );

    const nextPage = () => setPage(p => p + 1);
    const prevPage = () => setPage(p => Math.max(1, p - 1));

    return (
        <div>
            <Show when={usersQuery.data}>
                {(data) => (
                    <>
                        <For each={data().users}>
                            {(user) => <UserCard user={user} />}
                        </For>

                        <div>
                            <button onClick={prevPage} disabled={page() === 1}>
                                Previous
                            </button>
                            <span>Page {page()}</span>
                            <button onClick={nextPage}>
                                Next
                            </button>
                        </div>
                    </>
                )}
            </Show>
        </div>
    );
};
```

## Infinite Queries

```tsx
import { createInfiniteQuery } from "@tanstack/solid-query";

const InfiniteUserList: Component = () => {
    const usersQuery = createInfiniteQuery(() => ({
        queryKey: ["users", "infinite"],
        queryFn: ({ pageParam = 1 }) =>
            api.get<{ users: User[]; nextPage?: number }>(
                `/api/users?page=${pageParam}&limit=20`
            ),
        getNextPageParam: (lastPage) => lastPage.nextPage,
        initialPageParam: 1,
    }));

    return (
        <div>
            <For each={usersQuery.data?.pages}>
                {(page) => (
                    <For each={page.users}>
                        {(user) => <UserCard user={user} />}
                    </For>
                )}
            </For>

            <Show when={usersQuery.hasNextPage}>
                <button
                    onClick={() => usersQuery.fetchNextPage()}
                    disabled={usersQuery.isFetchingNextPage}
                >
                    {usersQuery.isFetchingNextPage ? "Loading..." : "Load More"}
                </button>
            </Show>
        </div>
    );
};
```

## Error Handling

```tsx
import { ApiClientError, NetworkError } from "@services/api";

const DataComponent: Component = () => {
    const dataQuery = useApiQuery<Data>(["data"], "/api/data");

    const errorMessage = () => {
        const error = dataQuery.error;
        if (!error) return null;

        if (error instanceof ApiClientError) {
            if (error.status === 404) {
                return "Data not found";
            }
            if (error.status === 403) {
                return "You don't have permission to view this data";
            }
            return error.message;
        }

        if (error instanceof NetworkError) {
            return "Network error - please check your connection";
        }

        return "An unexpected error occurred";
    };

    return (
        <div>
            <Show when={dataQuery.error}>
                <ErrorMessage message={errorMessage()} />
                <button onClick={() => dataQuery.refetch()}>
                    Retry
                </button>
            </Show>
        </div>
    );
};
```

## Query Key Patterns

```tsx
// Constants for query keys
export const QUERY_KEYS = {
    // Single resource
    user: (id: string) => ["user", id] as const,
    release: (id: string) => ["release", id] as const,

    // Collections
    users: () => ["users"] as const,
    usersList: (filters: UserFilters) => ["users", "list", filters] as const,

    // Nested resources
    userReleases: (userId: string) => ["user", userId, "releases"] as const,
    userFavorites: (userId: string) => ["user", userId, "favorites"] as const,

    // Infinite
    usersInfinite: () => ["users", "infinite"] as const,
};

// Usage
const userQuery = useApiQuery(
    QUERY_KEYS.user(userId),
    `/api/users/${userId}`
);

const releasesQuery = useApiQuery(
    QUERY_KEYS.userReleases(userId),
    `/api/users/${userId}/releases`
);
```

## Global Loading State

```tsx
import { useIsFetching, useIsMutating } from "@tanstack/solid-query";

const GlobalLoadingIndicator: Component = () => {
    const isFetching = useIsFetching();
    const isMutating = useIsMutating();

    const isLoading = () => isFetching() > 0 || isMutating() > 0;

    return (
        <Show when={isLoading()}>
            <div class={styles.globalLoader}>
                <Spinner />
            </div>
        </Show>
    );
};
```

## Summary

**Key Patterns:**
1. Use `@tanstack/solid-query` (NOT React Query)
2. Generic hooks for all HTTP methods
3. Query keys as constants
4. Optimistic updates for better UX
5. Query invalidation for cache management
6. Dependent queries with `enabled` option
7. Error handling with custom error classes
8. Loading states for all async operations
9. Success/error toast messages
10. Type safety with TypeScript generics

This ensures consistent, performant, and user-friendly API integration.
