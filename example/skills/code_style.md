---
name: Code Style
description: |
  Basic code style guidelines. Use this as a reference when writing or reviewing code.
---

# Code Style Guidelines

## Core Principles

1. **Clarity over cleverness** - Write code that's easy to understand
2. **Consistency** - Follow established patterns in the codebase
3. **Simplicity** - Prefer simple solutions over complex ones

## Naming Conventions

- **Variables**: Use descriptive names (`user_count` not `uc`)
- **Functions**: Use verb phrases (`calculate_total`, `get_user`)
- **Classes**: Use noun phrases (`UserManager`, `OrderProcessor`)
- **Constants**: Use UPPER_CASE (`MAX_RETRIES`, `DEFAULT_TIMEOUT`)

## Code Organization

- Keep functions small and focused (one responsibility)
- Group related code together
- Use meaningful file and directory names

## Comments

- Prefer self-documenting code over comments
- Only add comments for non-obvious "why" explanations
- Keep comments up-to-date with code changes

## Example

```python
# Good - clear naming and structure
def calculate_order_total(items: list[Item]) -> float:
    subtotal = sum(item.price * item.quantity for item in items)
    tax = subtotal * TAX_RATE
    return subtotal + tax

# Bad - unclear names and structure
def calc(x):
    s = sum(i.p * i.q for i in x)
    t = s * 0.08
    return s + t
```
