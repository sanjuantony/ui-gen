# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components they want to create, Claude generates code, and the preview updates in real-time. The application supports both anonymous and authenticated users, with optional persistence via SQLite.

## Key Commands

### Development
- `npm run dev` - Start Next.js dev server with Turbopack on http://localhost:3000
- `npm run dev:daemon` - Run dev server in background, logs to logs.txt

### Build & Deployment
- `npm run build` - Create production build
- `npm run start` - Run production server

### Database
- `npm run setup` - Install dependencies, generate Prisma client, and run migrations
- `npm run db:reset` - Reset database (destructive operation)

### Code Quality
- `npm run lint` - Run ESLint to check code for style issues
- `npm test` - Run Vitest unit tests
- `npm test -- --watch` - Run tests in watch mode for development

### Environment Setup
Create a `.env` file in the root to add Anthropic API key:
```
ANTHROPIC_API_KEY=your-api-key-here
```
The app runs without an API key, using mock/static component generation instead of LLM-powered generation.

## Architecture Overview

### Core Stack
- **Framework**: Next.js 15 with App Router, React 19, TypeScript
- **Styling**: Tailwind CSS v4 with CSS variables
- **Database**: Prisma + SQLite (optional, for user authentication)
- **AI Integration**: Vercel AI SDK with @ai-sdk/anthropic (Claude Haiku)
- **UI Components**: shadcn/ui (Radix UI primitives) + Lucide icons
- **Editor**: Monaco Editor via @monaco-editor/react
- **Transform**: Babel (standalone) for JSX compilation

### Directory Structure

```
src/
├── app/                    # Next.js App Router pages & layouts
│   ├── page.tsx           # Home page - redirects to project or shows landing
│   ├── layout.tsx         # Root layout with fonts & metadata
│   ├── [projectId]/       # Dynamic project page
│   ├── api/chat/          # POST endpoint for AI streaming
│   └── main-content.tsx   # Main UI layout with resizable panels
├── components/
│   ├── chat/              # ChatInterface, MessageList, MessageInput
│   ├── editor/            # FileTree, CodeEditor for code viewing/editing
│   ├── preview/           # PreviewFrame - renders JSX in iframe
│   ├── auth/              # Auth forms & components
│   └── ui/                # shadcn/ui components (Button, Dialog, Tabs, etc)
├── lib/
│   ├── file-system.ts     # VirtualFileSystem class - in-memory file tree (no disk I/O)
│   ├── auth.ts            # JWT session management via cookies
│   ├── prisma.ts          # Prisma client singleton
│   ├── provider.ts        # getLanguageModel() - returns Claude model or MockLanguageModel
│   ├── contexts/
│   │   ├── file-system-context.tsx  # React Context for VirtualFileSystem state
│   │   └── chat-context.tsx         # React Context wrapping useChat hook
│   ├── tools/
│   │   ├── str-replace.ts           # AI tool: file editor commands (view, create, str_replace, insert)
│   │   └── file-manager.ts          # AI tool: file operations (move, delete, rename)
│   ├── prompts/
│   │   └── generation.tsx           # System prompt for Claude - instructions for component generation
│   ├── transform/
│   │   └── jsx-transformer.ts       # Babel compilation, import mapping, HTML generation for iframe
│   └── __tests__/                   # Vitest unit tests for utilities
├── actions/                         # Server actions for DB operations
│   ├── create-project.ts
│   ├── get-project.ts
│   ├── get-projects.ts
│   └── index.ts                     # getUser(), signup(), login()
├── hooks/                           # Custom React hooks
├── generated/                       # Generated files (Prisma client)
└── middleware.ts                    # Request auth verification for protected routes
```

### Data Model

**User** (Prisma): id, email, password (bcrypt), createdAt, updatedAt
**Project** (Prisma): id, name, userId (optional), messages (JSON string), data (JSON string), createdAt, updatedAt

Projects store serialized state:
- `messages`: Array of chat messages (from AI SDK)
- `data`: VirtualFileSystem serialized state (file tree + content)

### Data Flow

1. **Chat Input**: User types message → ChatProvider (useChat hook) sends to `/api/chat` endpoint
2. **AI Generation**: Claude Haiku responds with tool calls (str_replace_editor, file_manager)
3. **Tool Execution**: FileSystemProvider applies tool calls to VirtualFileSystem (in-memory)
4. **Preview Update**: PreviewFrame watches VirtualFileSystem via refreshTrigger, recompiles JSX, renders in iframe
5. **Persistence**: On finalization, chat saves project data to Prisma (if authenticated)

### Virtual File System (VirtualFileSystem class)

The app operates entirely on a virtual file system - no files written to disk. Key methods:
- `createFile(path, content)`, `deleteFile(path)` - file operations
- `updateFile(path, content)`, `readFile(path)` - content access
- `createDirectory(path)`, `listDirectory(path)` - directory operations
- `serialize()`, `deserialize()` - convert to/from JSON for storage
- Tool commands: `viewFile()`, `createFileWithParents()`, `replaceInFile()`, `insertInFile()`

### AI Integration

- **Model**: Claude Haiku via `@ai-sdk/anthropic` (model ID: "claude-haiku-4-5")
- **Tools**: Two tool functions available to Claude:
  - `str_replace_editor`: create/view/edit files with string replacement or line insertion
  - `file_manager`: move/delete/rename operations
- **Context Window**: Uses prompt caching (ephemeral) for the system prompt
- **Max Tokens**: 10,000 per request; max 40 steps (agentic loop) for actual Claude, 4 for mock
- **Mock Provider**: When `ANTHROPIC_API_KEY` is not set, MockLanguageModel generates static component examples

### JSX Compilation & Preview

**jsx-transformer.ts** handles:
1. **Babel Transform**: Compiles JSX/TSX to JS using Babel standalone
2. **Import Mapping**: Creates importMap object mapping module paths to their code:
   - Local imports (e.g., `@/components/Button`) → inline code
   - Third-party imports (react, tailwindcss, lucide-react) → CDN/browser globals
3. **HTML Generation**: Wraps compiled code in `<script>` tags + Babel runtime in iframe
4. **Error Handling**: Catches missing imports, syntax errors, displays in iframe

**PreviewFrame** component:
- Watches refreshTrigger from FileSystemContext to know when to recompile
- Looks for entry point: `/App.jsx`, `/App.tsx`, `/index.jsx`, etc.
- Injects compiled code + import map into iframe sandbox
- Displays errors in UI if compilation or file loading fails

### Authentication

- **Session**: JWT-based, stored in httpOnly cookie (name: "auth-token", 7-day expiry)
- **Protected Routes**: `/api/projects/*` and `/api/filesystem/*` require valid session
- **Middleware**: `middleware.ts` verifies JWT on requests to protected paths
- **Anonymous Mode**: Users can generate components without signing up; data stored in LocalStorage via `anon-work-tracker.ts`

### Component Generation Prompt

System prompt (in `lib/prompts/generation.tsx`) instructs Claude to:
- Create React components (hooks, functional components allowed)
- Always create `/App.jsx` as the root entry point
- Use Tailwind CSS classes, not hardcoded styles
- Import from `@/` alias for local files
- No HTML files needed (virtual FS only)
- Keep responses brief unless user asks for summaries

## Testing

**Framework**: Vitest with jsdom environment for React/DOM testing
**Test Files**: Located in `__tests__` subdirectories parallel to source (e.g., `lib/__tests__/file-system.test.ts`)
**Coverage**: Unit tests for VirtualFileSystem, React contexts, JSX transformer, UI components

Example test command:
```bash
npm test -- file-system.test.ts  # Run single test file
```

## Important Notes & Conventions

### Path Aliases
Use `@/` prefix for imports (defined in `tsconfig.json` and `components.json`):
- `@/lib/*` → `src/lib/*`
- `@/components/*` → `src/components/*`
- `@/hooks/*` → `src/hooks/*`

### Node.js Compatibility
- `node-compat.cjs` loaded via NODE_OPTIONS in npm scripts to handle Node.js 25+ localStorage globals
- Required for SSR guard checks in dependencies

### State Management
- **Client State**: React Context (FileSystemProvider, ChatProvider) - prefer for UI/FS state
- **Server State**: Prisma + database - only for authenticated user projects
- **Transient State**: LocalStorage (anon-work-tracker) - tracks anonymous user work

### shadcn/ui Components
Using the "new-york" style with CSS variables. Components imported from `@/components/ui`:
- `Button`, `Dialog`, `Input`, `Textarea`, `Tabs`, `Separator`, `ScrollArea`, `Popover`, `Label`
- Style config in `components.json` (aliases, tailwind config)

### Error Handling
- Chat errors logged to console but don't block UX
- Preview errors displayed inline in iframe
- API errors caught in onError callback of streamText
- File system operations return null/false on failure with descriptive messages

### Comments
Use comments sparingly. Only comment complex or non-obvious code.

### ESLint & Type Safety
- Uses Next.js ESLint config (extends "next")
- TypeScript strict mode enabled
- No allowJs outside React context files

### Database Migrations
Located in `prisma/migrations/`. After schema changes:
```bash
npx prisma migrate dev --name <migration-name>
```
For development reset:
```bash
npm run db:reset
```

