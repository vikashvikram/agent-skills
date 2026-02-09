# Agent Skills

A collection of AI agent skills for building modern web applications.

## Available Skills

| Skill | Description |
|-------|-------------|
| [react-typescript-app](./react-typescript-app) | Build React applications with TypeScript, Material-UI, and modern best practices |
| [nodejs-typescript-app](./nodejs-typescript-app) | Build Node.js backend applications with TypeScript, Express.js, and layered architecture |
| [java-spring-boot-app](./java-spring-boot-app) | Build Java Spring Boot applications with JPA, validation, and layered architecture |

## Installation

Install all skills:

```bash
npx skills add vikash-singh/agent-skills
```

Or install individual skills by specifying the skill name after installation.

## Compatibility

These skills work with:
- Cursor
- Claude Code
- Cline
- Windsurf
- VS Code (with compatible extensions)
- And other AI coding assistants that support the skills format

## What's Included

### React TypeScript App

- Project structure patterns (api/, shared/, features/, types/)
- TypeScript configuration with path aliases
- Centralized type definitions
- API client abstraction pattern
- Custom hooks patterns
- ESLint best practices (optional chaining, replaceAll, object lookups)
- Material-UI styling guidelines
- Accessibility requirements
- Testing patterns with Jest and React Testing Library

### Node.js TypeScript App

- Layered architecture (routes/, services/, utils/, middleware/)
- Express.js setup and middleware patterns
- Async error handling
- Service layer for business logic
- Centralized error handling middleware
- Logging with Winston
- Database helpers pattern
- File upload handling with Multer
- Input validation patterns
- Testing patterns with Jest and Supertest

### Java Spring Boot App

- Layered architecture (controller/, service/, repository/, domain/, dto/)
- REST controller patterns with proper HTTP status codes
- Service layer with transactions and business logic
- JPA entities with relationships and auditing
- Java Records for DTOs with Bean Validation
- Custom validation annotations
- Spring Data JPA repositories
- Flyway database migrations
- Testcontainers for integration testing
- Structured logging patterns

## Usage

Once installed, these skills will automatically be applied when you:

- Create new React or Node.js projects
- Ask about project structure or architecture
- Write components, hooks, routes, or services
- Set up TypeScript configuration
- Write tests

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## License

MIT
