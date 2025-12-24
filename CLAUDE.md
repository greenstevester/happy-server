# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

This guide OVERRIDES any default behaviors and MUST be followed exactly.

## Project Overview

**Name**: happy-server  
**Repository**: https://github.com/slopus/happy-server.git  
**License**: MIT  
**Language**: TypeScript  
**Runtime**: Node.js 20  
**Framework**: Fastify with opinionated architecture  

## Core Technology Stack

- **Runtime**: Node.js 20
- **Language**: TypeScript (strict mode enabled)
- **Web Framework**: Fastify 5
- **Database**: PostgreSQL with Prisma ORM
- **Validation**: Zod
- **HTTP Client**: Axios
- **Real-time**: Socket.io
- **Cache/Pub-Sub**: Redis (via ioredis)
- **Testing**: Vitest
- **Package Manager**: Yarn (not npm)

## Development Environment

### Commands
- `yarn dev` - Start development server (kills existing process on port 3005, loads .env and .env.dev)
- `yarn build` - TypeScript type checking only (tsc --noEmit)
- `yarn start` - Start production server
- `yarn test` - Run all tests
- `yarn test sources/utils/lru.spec.ts` - Run a single test file
- `yarn migrate` - Run Prisma migrations (uses .env.dev)
- `yarn generate` - Generate Prisma client
- `yarn db` - Start local PostgreSQL in Docker
- `yarn redis` - Start local Redis in Docker
- `yarn s3` - Start local MinIO (S3-compatible) in Docker

### Environment Requirements
- FFmpeg installed (for media processing)
- Python3 installed
- PostgreSQL database
- Redis (for event bus and caching)

## Code Style and Structure

### General Principles
- Use 4 spaces for tabs (not 2 spaces)
- Write concise, technical TypeScript code with accurate examples
- Use functional and declarative programming patterns; avoid classes
- Prefer iteration and modularization over code duplication
- Use descriptive variable names with auxiliary verbs (e.g., isLoading, hasError)
- All sources must be imported using "@/" prefix (e.g., `import "@/utils/log"`)
- Always use absolute imports
- Prefer interfaces over types
- Avoid enums; use maps instead
- Use strict mode in TypeScript for better type safety

### Folder Structure
```
/sources
├── main.ts               # Entry point - initializes storage, auth, starts API
├── context.ts            # User context wrapper
├── types.ts              # Shared TypeScript types (incl. Prisma JSON types)
├── /app                  # Application-specific logic
│   ├── /api              # Fastify API server
│   │   ├── api.ts        # Server setup, route registration
│   │   ├── socket.ts     # Socket.io setup
│   │   ├── /routes       # REST endpoints (sessionRoutes.ts, authRoutes.ts, etc.)
│   │   ├── /socket       # WebSocket handlers
│   │   └── /utils        # Auth, monitoring, error handling
│   ├── /auth             # Authentication module
│   ├── /events           # eventRouter.ts - WebSocket event routing
│   ├── /session          # Session operations (sessionDelete.ts)
│   ├── /social           # Friend/relationship operations
│   ├── /feed             # Feed operations
│   ├── /kv               # Key-value store operations
│   └── /presence         # Activity tracking, session cache
├── /modules              # Reusable modules (non-app logic)
│   ├── encrypt.ts        # Encryption helpers
│   └── github.ts         # GitHub API integration
├── /storage              # Database and storage layer
│   ├── db.ts             # Prisma client
│   ├── redis.ts          # Redis client
│   ├── inTx.ts           # Transaction wrapper with afterTx callbacks
│   ├── seq.ts            # Sequence number allocation
│   └── files.ts          # S3/MinIO file storage
└── /utils                # Low-level utilities
```

### Naming Conventions
- Use lowercase with dashes for directories (e.g., components/auth-wizard)
- When writing utility functions, always name file and function the same way
- Test files should have ".spec.ts" suffix

## Tool Usage

### Web Search and Fetching
- When in doubt, use web tool to get answers from the web
- Search web when you have some failures

### File Operations
- NEVER create files unless they're absolutely necessary
- ALWAYS prefer editing existing files to creating new ones
- NEVER proactively create documentation files (*.md) or README files unless explicitly requested

## Utilities

### Writing Utility Functions
1. Always name file and function the same way for easy discovery
2. Utility functions should be modular and not too complex
3. Always write tests for utility functions BEFORE writing the code
4. Iterate implementation and tests until the function works as expected
5. Always write documentation for utility functions

## Modules

### Module Guidelines
- Modules are bigger than utility functions and abstract away complexity
- Each module should have a dedicated directory
- Modules usually don't have application-specific logic
- Modules can depend on other modules, but not on application-specific logic
- Prefer to write code as modules instead of application-specific code

### When to Use Modules
- When integrating with external services
- When abstracting complexity of some library
- When implementing related groups of functions (math, date, etc.)

### Known Modules
- **ai**: AI wrappers to interact with AI services
- **eventbus**: Event bus to send and receive events between modules and applications
- **lock**: Simple lock to synchronize access to resources in the whole cluster
- **media**: Tools to work with media files

## Applications

- Applications contain application-specific logic
- Applications have the most complexity; other parts should assist by reducing complexity
- When using prompts, write them to "_prompts.ts" file relative to the application

## Database

### Prisma Usage
- Prisma is used as ORM
- Use "inTx" to wrap database operations in transactions
- Do not update schema without absolute necessity
- For complex fields, use "Json" type
- NEVER DO MIGRATION YOURSELF. Only run yarn generate when new types needed

### Typed JSON Fields
Use `/// [TypeName]` comments in schema.prisma to enable typed JSON fields:
```prisma
/// [SessionMessageContent]
content   Json
```
Types are defined in `@/types.ts` and generated via `prisma-json-types-generator`.

## Events

### Event Architecture
The server uses an EventRouter (`@/app/events/eventRouter`) for real-time updates:

**Event Types:**
- `UpdateEvent` - Persistent updates with sequence numbers (new-session, new-message, update-session, etc.)
- `EphemeralEvent` - Transient status (activity, machine-status, usage) - not persisted

**Connection Scopes:**
- `user-scoped` - Mobile/web apps that need all user data
- `session-scoped` - CLI connections bound to a specific session
- `machine-scoped` - Daemon connections bound to a specific machine

**Recipient Filters:**
- `all-interested-in-session` - Session-scoped (matching) + user-scoped
- `user-scoped-only` - Only mobile/web apps
- `machine-scoped-only` - User-scoped + specific machine
- `all-user-authenticated-connections` - Everyone

**Sending Updates:**
- Use `afterTx()` to emit events after transaction commits successfully
- Use `eventRouter.emitUpdate()` for persistent updates
- Use `eventRouter.emitEphemeral()` for transient status
- Updates include sequence numbers via `allocateUserSeq()` for client ordering

## Testing

- Write tests using Vitest
- Test files should be named the same as source files with ".spec.ts" suffix
- For utility functions, write tests BEFORE implementation

## API Development

- API server entry: `/sources/app/api/api.ts`
- Routes: `/sources/app/api/routes/`
- Socket handlers: `/sources/app/api/socket/`

### Route Pattern
```typescript
app.post('/v1/endpoint', {
    schema: {
        body: z.object({ ... }),
        params: z.object({ ... })
    },
    preHandler: app.authenticate  // Adds request.userId
}, async (request, reply) => {
    const userId = request.userId;
    // ...
});
```

### Key Principles
- Always validate inputs using Zod schemas
- **Idempotency**: All operations must be idempotent - clients retry automatically
- Use `preHandler: app.authenticate` for authenticated routes
- Server runs on port 3005 (configurable via PORT env var)

## Docker Deployment

The project includes a multi-stage Dockerfile:
1. Builder stage: Installs dependencies and builds the application
2. Runner stage: Minimal runtime with only necessary files
3. Exposes port 3000
4. Requires FFmpeg and Python3 in the runtime

## Important Reminders

1. Do what has been asked; nothing more, nothing less
2. NEVER create files unless they're absolutely necessary for achieving your goal
3. ALWAYS prefer editing an existing file to creating a new one
4. NEVER proactively create documentation files (*.md) or README files unless explicitly requested
5. Use 4 spaces for tabs (not 2 spaces)
6. Use yarn instead of npm for all package management

## Debugging Notes

### Remote Logging Setup
- Use `DANGEROUSLY_LOG_TO_SERVER_FOR_AI_AUTO_DEBUGGING=true` env var to enable
- Server logs to `.logs/` directory with timestamped files (format: `MM-DD-HH-MM-SS.log`)
- Mobile and CLI send logs to `/logs-combined-from-cli-and-mobile-for-simple-ai-debugging` endpoint

### Common Issues & Tells

#### Socket/Connection Issues
- **Tell**: "Sending update to user-scoped connection" but mobile not updating
- **Tell**: Multiple "User disconnected" messages indicate socket instability
- **Tell**: "Response from the Engine was empty" = Prisma database connection lost

#### Auth Flow Debugging
- CLI must hit `/v1/auth/request` to create auth request
- Mobile scans QR and hits `/v1/auth/response` to approve
- **Tell**: 404 on `/v1/auth/response` = server likely restarted/crashed
- **Tell**: "Auth failed - user not found" = token issue or user doesn't exist

#### Session Creation Flow
- Sessions created via POST `/v1/sessions` with tag-based deduplication
- Server emits "new-session" update to all user connections
- **Tell**: Sessions created but not showing = mobile app not processing updates
- **Tell**: "pathname /" in mobile logs = app stuck at root screen

#### Environment Variables
- CLI: Use `yarn dev:local-server` (NOT `yarn dev`) to load `.env.dev-local-server`
- Server: Use `yarn dev` to start with proper env files
- **Tell**: Wrong server URL = check `HAPPY_SERVER_URL` env var
- **Tell**: Wrong home dir = check `HAPPY_HOME_DIR` (should be `~/.happy-dev` for local)

### Quick Diagnostic Commands

#### IMPORTANT: Always Start Debugging With These
```bash
# 1. CHECK CURRENT TIME - Logs use local time, know what's current!
date

# 2. CHECK LATEST LOG FILES - Server creates new logs on restart
ls -la .logs/*.log | tail -5

# 3. VERIFY YOU'RE LOOKING AT CURRENT LOGS
# Server logs are named: MM-DD-HH-MM-SS.log (month-day-hour-min-sec)
# If current time is 13:45 and latest log is 08-15-10-57-02.log from 10:57,
# that log started 3 hours ago but may still be active!
tail -1 .logs/[LATEST_LOG_FILE]  # Check last entry timestamp
```

#### Common Debugging Patterns
```bash
# Check server logs for errors
tail -100 .logs/*.log | grep -E "(error|Error|ERROR|failed|Failed)"

# Monitor session creation
tail -f .logs/*.log | grep -E "(new-session|Session created)"

# Check active connections
tail -100 .logs/*.log | grep -E "(Token verified|User connected|User disconnected)"

# See what endpoints are being hit
tail -100 .logs/*.log | grep "incoming request"

# Debug socket real-time updates
tail -500 .logs/*.log | grep -A 2 -B 2 "new-session" | tail -30
tail -200 .logs/*.log | grep -E "(websocket|Socket.*connected|Sending update)" | tail -30

# Track socket events from mobile client
tail -300 .logs/*.log | grep "remote-log.*mobile" | grep -E "(SyncSocket|handleUpdate)" | tail -20

# Monitor session creation flow end-to-end
tail -500 .logs/*.log | grep "session-create" | tail -20
tail -500 .logs/*.log | grep "cmed556s4002bvb2020igg8jf" -A 3 -B 3  # Replace with actual session ID

# Check auth flow for sessions API
tail -300 .logs/*.log | grep "auth-decorator.*sessions" | tail -10

# Debug machine registration and online status
tail -500 .logs/*.log | grep -E "(machine-alive|machine-register|update-machine)" | tail -20
tail -500 .logs/*.log | grep "GET /v1/machines" | tail -10
tail -500 .logs/*.log | grep "POST /v1/machines" | tail -10

# Check what mobile app is seeing
tail -500 .logs/*.log | grep "📊 Storage" | tail -20
tail -500 .logs/*.log | grep "applySessions.*active" | tail -10
```

#### Time Format Reference
- **CLI logs**: `[HH:MM:SS.mmm]` in local time (e.g., `[13:45:23.738]`)
- **Server logs**: Include both `time` (Unix ms) and `localTime` (HH:MM:ss.mmm)
- **Mobile logs**: Sent with `timestamp` in UTC, converted to `localTime` on server
- **All consolidated logs**: Have `localTime` field for easy correlation
- When writing a some operations on db, like adding friend, sending a notification - always create a dedicated file in relevant subfolder of the @sources/app/ folder. Good example is "friendAdd", always prefix with an entity type, then action that should be performed.
- Never create migrations yourself, it is can be done only by human
- Do not return stuff from action functions "just in case", only essential
- Do not add logging when not asked
- do not run non-transactional things (like uploadign files) in transactions
- After writing an action - add a documentation comment that explains logic, also keep it in sync.
- always use github usernames
- Always use privacyKit.decodeBase64 and privacyKit.encodeBase64 from privacy-kit instead of using buffer