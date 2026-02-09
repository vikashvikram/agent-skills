---
name: nodejs-typescript-app
description: Build and structure Node.js backend applications with TypeScript, Express.js, and layered architecture. Use when creating APIs, routes, services, middleware, database helpers, or when the user asks about Node.js project structure, backend architecture, or Express patterns.
---

# Node.js TypeScript Application

Patterns and best practices for building production-ready Node.js backends with TypeScript and Express.js.

## Project Structure

```
backend/
├── routes/                 # Express route handlers (HTTP layer)
│   ├── index.ts            # Route aggregator
│   ├── auth.ts
│   ├── users.ts
│   └── products.ts
├── services/               # Business logic layer
│   ├── authService.ts
│   ├── userService.ts
│   └── productService.ts
├── middleware/             # Express middleware
│   ├── errorHandler.ts
│   ├── auth.ts
│   └── validation.ts
├── utils/                  # Shared utilities
│   ├── logger.ts
│   ├── db.ts
│   └── helpers.ts
├── types/                  # TypeScript definitions
│   └── index.ts
├── config/                 # Configuration
│   └── index.ts
├── index.ts                # Application entry point
├── package.json
└── tsconfig.json
```

## TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "sourceMap": true
  },
  "include": ["./**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

## Package Scripts

```json
{
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "nodemon --exec ts-node index.ts",
    "lint": "eslint . --ext .ts",
    "test": "jest"
  }
}
```

## Application Entry Point

```typescript
// index.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import routes from './routes';
import { errorHandler } from './middleware/errorHandler';
import { logger } from './utils/logger';
import config from './config';

const app = express();

// Security middleware
app.use(helmet());
app.use(cors());

// Body parsing
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// API routes
app.use('/api', routes);

// Health check
app.get('/health', (_, res) => res.json({ status: 'ok' }));

// Error handler (must be last)
app.use(errorHandler);

app.listen(config.port, () => {
  logger.info(`Server running on port ${config.port}`);
});

export default app;
```

## Route Layer

Routes handle HTTP concerns only - delegate business logic to services.

### Route Aggregator

```typescript
// routes/index.ts
import { Router } from 'express';
import authRoutes from './auth';
import userRoutes from './users';
import productRoutes from './products';

const router = Router();

router.use('/auth', authRoutes);
router.use('/users', userRoutes);
router.use('/products', productRoutes);

export default router;
```

### Route Handler Pattern

```typescript
// routes/users.ts
import { Router, Request, Response, NextFunction } from 'express';
import { userService } from '../services/userService';
import { asyncHandler } from '../middleware/asyncHandler';
import { validateBody } from '../middleware/validation';
import { createUserSchema } from '../schemas/user';

const router = Router();

// GET /api/users
router.get('/', asyncHandler(async (req: Request, res: Response) => {
  const users = await userService.getAll();
  res.json(users);
}));

// GET /api/users/:id
router.get('/:id', asyncHandler(async (req: Request, res: Response) => {
  const user = await userService.getById(req.params.id);
  if (!user) {
    return res.status(404).json({ error: 'User not found' });
  }
  res.json(user);
}));

// POST /api/users
router.post('/',
  validateBody(createUserSchema),
  asyncHandler(async (req: Request, res: Response) => {
    const user = await userService.create(req.body);
    res.status(201).json(user);
  })
);

// DELETE /api/users/:id
router.delete('/:id', asyncHandler(async (req: Request, res: Response) => {
  await userService.delete(req.params.id);
  res.json({ message: 'User deleted successfully' });
}));

export default router;
```

## Service Layer

Services contain business logic, separate from HTTP concerns.

```typescript
// services/userService.ts
import { db } from '../utils/db';
import { AppError } from '../middleware/errorHandler';
import type { User, CreateUserInput } from '../types';

export const userService = {
  async getAll(): Promise<User[]> {
    return db.query<User>('SELECT * FROM users ORDER BY created_at DESC');
  },

  async getById(id: string): Promise<User | null> {
    const [user] = await db.query<User>('SELECT * FROM users WHERE id = ?', [id]);
    return user || null;
  },

  async create(input: CreateUserInput): Promise<User> {
    // Business validation
    const existing = await db.query<User>('SELECT id FROM users WHERE email = ?', [input.email]);
    if (existing.length > 0) {
      throw new AppError('Email already exists', 409);
    }

    const id = crypto.randomUUID();
    await db.execute(
      'INSERT INTO users (id, name, email, created_at) VALUES (?, ?, ?, ?)',
      [id, input.name, input.email, new Date().toISOString()]
    );

    return this.getById(id) as Promise<User>;
  },

  async delete(id: string): Promise<void> {
    const user = await this.getById(id);
    if (!user) {
      throw new AppError('User not found', 404);
    }
    await db.execute('DELETE FROM users WHERE id = ?', [id]);
  },
};
```

## Middleware

### Async Handler

Wraps async route handlers to catch errors automatically.

```typescript
// middleware/asyncHandler.ts
import { Request, Response, NextFunction } from 'express';

type AsyncFunction = (req: Request, res: Response, next: NextFunction) => Promise<unknown>;

export const asyncHandler = (fn: AsyncFunction) =>
  (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
```

### Error Handler

Centralized error handling with custom error class.

```typescript
// middleware/errorHandler.ts
import { Request, Response, NextFunction } from 'express';
import { logger } from '../utils/logger';

export class AppError extends Error {
  constructor(
    message: string,
    public statusCode: number = 500,
    public isOperational: boolean = true
  ) {
    super(message);
    this.name = 'AppError';
    Error.captureStackTrace(this, this.constructor);
  }
}

export const errorHandler = (
  err: Error | AppError,
  req: Request,
  res: Response,
  _next: NextFunction
): void => {
  const statusCode = err instanceof AppError ? err.statusCode : 500;
  const message = err.message || 'Internal server error';

  logger.error(`${req.method} ${req.path} - ${statusCode} - ${message}`, {
    stack: err.stack,
    body: req.body,
  });

  res.status(statusCode).json({
    error: message,
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
  });
};
```

### Input Validation

```typescript
// middleware/validation.ts
import { Request, Response, NextFunction } from 'express';
import { ZodSchema } from 'zod';

export const validateBody = (schema: ZodSchema) =>
  (req: Request, res: Response, next: NextFunction) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({
        error: 'Validation failed',
        details: result.error.flatten().fieldErrors,
      });
    }
    req.body = result.data;
    next();
  };
```

## Utilities

### Logger

```typescript
// utils/logger.ts
import winston from 'winston';
import config from '../config';

export const logger = winston.createLogger({
  level: config.logLevel,
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.printf(({ level, message, timestamp, stack }) =>
      `${timestamp} [${level.toUpperCase()}]: ${message}${stack ? `\n${stack}` : ''}`
    )
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'server.log' }),
  ],
});
```

### Database Helper

```typescript
// utils/db.ts
import { logger } from './logger';

// Generic database interface - implement for your DB (PostgreSQL, MySQL, SQLite, etc.)
export const db = {
  async query<T>(sql: string, params?: unknown[]): Promise<T[]> {
    logger.debug(`Query: ${sql}`, { params });
    // Implement with your database driver
    return [];
  },

  async execute(sql: string, params?: unknown[]): Promise<void> {
    logger.debug(`Execute: ${sql}`, { params });
    // Implement with your database driver
  },
};
```

### Configuration

```typescript
// config/index.ts
import dotenv from 'dotenv';
dotenv.config();

const config = {
  port: parseInt(process.env.PORT || '3001', 10),
  nodeEnv: process.env.NODE_ENV || 'development',
  logLevel: process.env.LOG_LEVEL || 'info',
  
  db: {
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT || '5432', 10),
    name: process.env.DB_NAME || 'app',
    user: process.env.DB_USER || 'postgres',
    password: process.env.DB_PASSWORD || '',
  },

  jwt: {
    secret: process.env.JWT_SECRET || 'change-me-in-production',
    expiresIn: process.env.JWT_EXPIRES_IN || '7d',
  },
};

export default config;
```

## Type Definitions

```typescript
// types/index.ts
export interface User {
  id: string;
  name: string;
  email: string;
  createdAt: string;
}

export interface CreateUserInput {
  name: string;
  email: string;
}

// Express extensions
declare global {
  namespace Express {
    interface Request {
      user?: User;
    }
  }
}

export {};
```

## File Uploads

```typescript
// routes/upload.ts
import { Router } from 'express';
import multer from 'multer';
import path from 'path';
import { asyncHandler } from '../middleware/asyncHandler';

const storage = multer.diskStorage({
  destination: './uploads',
  filename: (req, file, cb) => {
    const uniqueName = `${Date.now()}-${Math.round(Math.random() * 1E9)}${path.extname(file.originalname)}`;
    cb(null, uniqueName);
  },
});

const upload = multer({
  storage,
  limits: { fileSize: 50 * 1024 * 1024 }, // 50MB
  fileFilter: (req, file, cb) => {
    const allowed = ['.csv', '.xlsx', '.json', '.pdf'];
    const ext = path.extname(file.originalname).toLowerCase();
    cb(null, allowed.includes(ext));
  },
});

const router = Router();

router.post('/', upload.single('file'), asyncHandler(async (req, res) => {
  if (!req.file) {
    return res.status(400).json({ error: 'No file uploaded' });
  }
  res.json({
    message: 'File uploaded',
    filename: req.file.filename,
    size: req.file.size,
  });
}));

export default router;
```

## Testing

```typescript
// tests/routes/users.test.ts
import request from 'supertest';
import app from '../../index';
import { userService } from '../../services/userService';
import type { User } from '../../types';

jest.mock('../../services/userService');
const mockedService = userService as jest.Mocked<typeof userService>;

describe('GET /api/users', () => {
  beforeEach(() => jest.clearAllMocks());

  it('returns all users', async () => {
    const mockUsers: User[] = [
      { id: '1', name: 'John', email: 'john@test.com', createdAt: '2024-01-01' }
    ];
    mockedService.getAll.mockResolvedValue(mockUsers);

    const res = await request(app).get('/api/users');

    expect(res.status).toBe(200);
    expect(res.body).toEqual(mockUsers);
    expect(mockedService.getAll).toHaveBeenCalledTimes(1);
  });

  it('handles service errors', async () => {
    mockedService.getAll.mockRejectedValue(new Error('Database error'));

    const res = await request(app).get('/api/users');

    expect(res.status).toBe(500);
    expect(res.body).toHaveProperty('error');
  });
});
```

## Dependencies

```json
{
  "dependencies": {
    "express": "^4.x",
    "cors": "^2.x",
    "helmet": "^7.x",
    "dotenv": "^16.x",
    "winston": "^3.x",
    "zod": "^3.x"
  },
  "devDependencies": {
    "typescript": "^5.x",
    "@types/node": "^20.x",
    "@types/express": "^4.x",
    "@types/cors": "^2.x",
    "nodemon": "^3.x",
    "ts-node": "^10.x",
    "jest": "^29.x",
    "ts-jest": "^29.x",
    "supertest": "^6.x",
    "@types/supertest": "^6.x"
  }
}
```

## Best Practices Summary

1. **Layered architecture**: Routes → Services → Database
2. **Async handlers**: Always wrap async routes to catch errors
3. **Centralized errors**: Use AppError class with status codes
4. **Input validation**: Validate at route level before service
5. **Type everything**: Define types in `types/index.ts`
6. **Configuration**: Use environment variables with defaults
7. **Logging**: Log errors with context (method, path, body)
