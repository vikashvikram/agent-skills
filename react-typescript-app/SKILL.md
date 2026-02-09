---
name: react-typescript-app
description: Build and structure React applications with TypeScript and modern best practices. Use when creating React components, hooks, API layers, configuring TypeScript, writing tests, or when the user asks about React project structure, frontend architecture, or React patterns.
---

# React TypeScript Application

Patterns and best practices for building production-ready React applications with TypeScript.

## Project Structure

```
src/
├── api/                    # API client and endpoint modules
│   ├── client.ts           # Base API client with error handling
│   ├── index.ts            # Re-exports all API functions
│   └── [domain].ts         # Domain-specific endpoints (users.ts, products.ts)
├── components/             # Shared/common components
│   ├── ui/                 # Base UI components (Button, Modal, etc.)
│   └── layout/             # Layout components (Header, Sidebar)
├── features/               # Feature-based modules
│   └── [feature]/
│       ├── components/     # Feature-specific components
│       ├── hooks/          # Feature-specific hooks
│       └── index.ts
├── hooks/                  # Shared custom hooks
├── types/                  # Centralized TypeScript definitions
│   └── index.ts
├── utils/                  # Helper functions
├── constants/              # App constants and config
├── App.tsx
└── index.tsx
```

## TypeScript Configuration

Essential `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["DOM", "DOM.Iterable", "ES2021"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "module": "ESNext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "baseUrl": "src",
    "types": ["jest", "node"],
    "paths": {
      "@/*": ["./*"],
      "@/components/*": ["./components/*"],
      "@/hooks/*": ["./hooks/*"],
      "@/types/*": ["./types/*"]
    }
  },
  "include": ["src"],
  "exclude": ["node_modules", "build"]
}
```

**Key settings:**
- `lib: ["ES2021"]` - enables `replaceAll()` and other modern APIs
- `types: ["jest", "node"]` - includes test runner types
- `paths` - enables clean imports like `@/components/Button`

## Type Definitions

Centralize types in `types/index.ts`:

```typescript
// API response wrapper
export interface ApiResponse<T = unknown> {
  data?: T;
  message?: string;
  error?: string;
}

// Pagination
export interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  pageSize: number;
  totalPages: number;
}

// Common component props
export interface ModalProps {
  open: boolean;
  onClose: () => void;
}

// Domain types - always include all required fields
export interface User {
  id: string;
  name: string;      // Include display fields
  email: string;
  createdAt: string;
}
```

## API Layer

### Base Client

```typescript
// api/client.ts
export const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:3001';

export class ApiError extends Error {
  constructor(message: string, public status?: number) {
    super(message);
    this.name = 'ApiError';
  }
}

export const apiClient = {
  async get<T>(endpoint: string): Promise<T> {
    const res = await fetch(`${API_URL}${endpoint}`);
    if (!res.ok) throw new ApiError(await res.text(), res.status);
    return res.json();
  },

  async post<T>(endpoint: string, data?: unknown): Promise<T> {
    const res = await fetch(`${API_URL}${endpoint}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: data ? JSON.stringify(data) : undefined,
    });
    if (!res.ok) throw new ApiError(await res.text(), res.status);
    return res.json();
  },

  async delete<T>(endpoint: string): Promise<T> {
    const res = await fetch(`${API_URL}${endpoint}`, { method: 'DELETE' });
    if (!res.ok) throw new ApiError(await res.text(), res.status);
    return res.json();
  },
};
```

### Domain Endpoints

```typescript
// api/users.ts
import { apiClient } from './client';
import type { User } from '../types';

export const getUsers = (): Promise<User[]> => 
  apiClient.get('/api/users');

export const createUser = (data: Omit<User, 'id' | 'createdAt'>): Promise<User> =>
  apiClient.post('/api/users', data);

export const deleteUser = (id: string): Promise<{ message: string }> =>
  apiClient.delete(`/api/users/${id}`);
```

## Custom Hooks

```typescript
// hooks/useAsync.ts
import { useState, useCallback } from 'react';

interface UseAsyncReturn<T> {
  data: T | null;
  loading: boolean;
  error: string | null;
  execute: (...args: unknown[]) => Promise<T | undefined>;
}

export function useAsync<T>(asyncFn: (...args: unknown[]) => Promise<T>): UseAsyncReturn<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const execute = useCallback(async (...args: unknown[]) => {
    setLoading(true);
    setError(null);
    try {
      const result = await asyncFn(...args);
      setData(result);
      return result;
    } catch (err) {
      setError(err instanceof Error ? err.message : 'An error occurred');
    } finally {
      setLoading(false);
    }
  }, [asyncFn]);

  return { data, loading, error, execute };
}
```

## Code Quality & Best Practices

### Optional Chaining

```typescript
// ✅ Good - concise and safe
if (!user?.profile?.settings) return null;
const city = user?.address?.city ?? 'Unknown';

// ❌ Avoid - verbose and error-prone
if (!user || !user.profile || !user.profile.settings) return null;
const city = user && user.address && user.address.city ? user.address.city : 'Unknown';
```

### Nullish Coalescing

```typescript
// ✅ Good - only falls back on null/undefined
const count = value ?? 0;        // Preserves 0 and '' as valid values
const name = user.name ?? 'Anonymous';

// ❌ Avoid - || treats 0, '', false as falsy
const count = value || 0;        // Bug: replaces 0 with 0 (works), but intent unclear
```

### replaceAll() Over replace() with Global Flag

```typescript
// ✅ Good - semantic and clear intent
const clean = input.replaceAll(/[\\/]/g, '_');
const normalized = text.replaceAll(/\s+/g, ' ');

// ❌ Avoid - less clear intent
const clean = input.replace(/[\\/]/g, '_');
```

### Object Lookups Over Nested Ternaries

```typescript
// ✅ Good - readable, extensible, and maintainable
const statusColors: Record<string, string> = {
  success: 'green',
  warning: 'orange',
  error: 'red',
  info: 'blue',
};
const color = statusColors[status] || 'gray';

// Also good for functions
const handlers: Record<string, () => void> = {
  save: handleSave,
  delete: handleDelete,
  cancel: handleCancel,
};
handlers[action]?.();

// ❌ Avoid - hard to read and extend
const color = status === 'success' ? 'green'
  : status === 'warning' ? 'orange'
  : status === 'error' ? 'red'
  : status === 'info' ? 'blue' : 'gray';
```

### Avoid Unused Variables

```typescript
// ✅ Good - prefix with underscore for intentionally unused
const [_, setCount] = useState(0);
array.map((_item, index) => index);

// ✅ Good - destructure only what you need
const { name, email } = user;  // Don't destructure id if not used

// ❌ Avoid - declared but never used
const unused = getValue();  // ESLint: 'unused' is declared but never used
```

### Strict Equality

```typescript
// ✅ Good - strict equality
if (value === null) { }
if (status !== 'active') { }

// ❌ Avoid - loose equality (type coercion)
if (value == null) { }  // Also matches undefined
if (status != 'active') { }
```

### Array Methods Over Loops

```typescript
// ✅ Good - functional, declarative
const activeUsers = users.filter(u => u.isActive);
const names = users.map(u => u.name);
const hasAdmin = users.some(u => u.role === 'admin');
const allVerified = users.every(u => u.verified);

// ❌ Avoid - imperative loops for simple transformations
const activeUsers = [];
for (let i = 0; i < users.length; i++) {
  if (users[i].isActive) activeUsers.push(users[i]);
}
```

### Early Returns

```typescript
// ✅ Good - guard clauses reduce nesting
function processUser(user: User | null): string {
  if (!user) return 'No user';
  if (!user.isActive) return 'Inactive';
  if (!user.hasPermission) return 'No permission';
  
  return `Welcome, ${user.name}`;
}

// ❌ Avoid - deep nesting
function processUser(user: User | null): string {
  if (user) {
    if (user.isActive) {
      if (user.hasPermission) {
        return `Welcome, ${user.name}`;
      } else {
        return 'No permission';
      }
    } else {
      return 'Inactive';
    }
  } else {
    return 'No user';
  }
}
```

### Accessibility

```typescript
// Hidden inputs still need accessible names
<input
  type="file"
  style={{ display: 'none' }}
  aria-label="Upload file"
  onChange={handleChange}
/>

// Buttons with only icons need labels
<IconButton aria-label="Delete item">
  <DeleteIcon />
</IconButton>

// Form elements need labels
<TextField
  label="Email address"  // MUI: visible label
  // or
  inputProps={{ 'aria-label': 'Email address' }}  // Screen reader only
/>
```

### Consistent Imports

```typescript
// ✅ Good - group and order imports
// 1. React/external libraries
import React, { useState, useEffect } from 'react';
import { Box, Button } from '@mui/material';

// 2. Internal absolute imports
import { useAuth } from '@/hooks/useAuth';
import { User } from '@/types';

// 3. Relative imports
import { UserCard } from './UserCard';
import styles from './styles.module.css';

// ❌ Avoid - mixed, unorganized imports
import { UserCard } from './UserCard';
import React from 'react';
import { useAuth } from '@/hooks/useAuth';
import { Box } from '@mui/material';
```

## Component Patterns

### Modal with Loading State

```typescript
interface SaveModalProps {
  open: boolean;
  onClose: () => void;
  onSave: (name: string) => Promise<void>;
  defaultValue?: string;
}

export const SaveModal: React.FC<SaveModalProps> = ({
  open, onClose, onSave, defaultValue = ''
}) => {
  const [value, setValue] = useState(defaultValue);
  const [saving, setSaving] = useState(false);

  // Reset on open
  useEffect(() => {
    if (open) setValue(defaultValue);
  }, [open, defaultValue]);

  const handleSave = async () => {
    if (!value.trim()) return;
    setSaving(true);
    try {
      await onSave(value.trim());
      onClose();
    } finally {
      setSaving(false);
    }
  };

  return (
    <Modal open={open} onClose={() => !saving && onClose()}>
      {/* Disable close during save */}
    </Modal>
  );
};
```

### Controlled List with Actions

```typescript
interface ItemListProps<T> {
  items: T[];
  onDelete: (id: string) => void;
  onSelect: (item: T) => void;
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string;
}

export function ItemList<T>({ items, onDelete, onSelect, renderItem, keyExtractor }: ItemListProps<T>) {
  return (
    <List>
      {items.map((item) => (
        <ListItem key={keyExtractor(item)} onClick={() => onSelect(item)}>
          {renderItem(item)}
          <IconButton onClick={(e) => { e.stopPropagation(); onDelete(keyExtractor(item)); }}>
            <DeleteIcon />
          </IconButton>
        </ListItem>
      ))}
    </List>
  );
}
```

## Testing Patterns

```typescript
import { renderHook, act, waitFor } from '@testing-library/react';
import { useUsers } from '../useUsers';
import * as api from '../../api';
import type { User } from '../../types';

jest.mock('../../api');
const mockedApi = api as jest.Mocked<typeof api>;

describe('useUsers', () => {
  beforeEach(() => jest.clearAllMocks());

  it('fetches users on mount', async () => {
    const mockUsers: User[] = [
      { id: '1', name: 'John', email: 'john@example.com', createdAt: '2024-01-01' }
    ];
    mockedApi.getUsers.mockResolvedValueOnce(mockUsers);

    const { result } = renderHook(() => useUsers());

    await waitFor(() => {
      expect(result.current.users).toEqual(mockUsers);
    });
  });
});
```

**Testing rules:**
1. Import types from centralized `types/` - not local interfaces
2. Mock data must match full type definitions
3. Use `jest.Mocked<typeof module>` for type-safe mocks

## Styling Guidelines

### Use `sx` Prop for MUI Components

```typescript
<Box sx={{ display: 'flex', gap: 2, p: 2 }}>
  <Typography sx={{ fontWeight: 600, color: 'primary.main' }}>
    Title
  </Typography>
</Box>
```

### When Inline Styles Are OK

1. **Dynamic values** based on state: `style={{ opacity: loading ? 0.5 : 1 }}`
2. **Hidden elements**: `style={{ display: 'none' }}`
3. **Computed values**: `style={{ width: `${percent}%` }}`

## Dependencies

```json
{
  "dependencies": {
    "react": "^18.x",
    "@mui/material": "^5.x",
    "@emotion/react": "^11.x",
    "@emotion/styled": "^11.x"
  },
  "devDependencies": {
    "typescript": "^5.x",
    "@types/react": "^18.x",
    "@types/jest": "^29.x",
    "@testing-library/react": "^14.x"
  }
}
```

Use `npm install --legacy-peer-deps` if encountering peer dependency conflicts.
