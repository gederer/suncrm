# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **SunCRM** project - a Next.js application with Convex backend for full-stack development. The stack includes:

- **Frontend**: Next.js 15 with React 19, TypeScript, Tailwind CSS v4
- **Backend**: Convex (real-time database, server functions)
- **Development**: npm-run-all for parallel development servers

## Development Commands

### Start Development
```bash
npm run dev
```
Runs both frontend (Next.js) and backend (Convex) in parallel. The predev script ensures Convex is ready and opens the dashboard.

### Individual Services
```bash
npm run dev:frontend  # Next.js dev server only
npm run dev:backend   # Convex dev server only
```

### Build & Production
```bash
npm run build  # Build Next.js app
npm run start  # Start production server
```

### Code Quality
```bash
npm run lint   # ESLint with Next.js and TypeScript configs
```

## Architecture

### Frontend Structure
- **App Router**: Using Next.js 13+ app directory structure
- **Components**: React components in `/components` directory
- **Layout**: Root layout with Convex provider and font configuration
- **Styling**: Tailwind CSS v4 with Geist fonts

### Backend Structure (Convex)
- **Schema**: Database schema in `convex/schema.ts`
- **Functions**: Server functions in `convex/` directory
- **Generated**: Auto-generated API types in `convex/_generated/`

### Key Patterns
- **Convex Integration**: Client provider wraps the app for real-time data
- **TypeScript**: Strict typing with path aliases (`@/*` → `./`)
- **Environment**: Uses `NEXT_PUBLIC_CONVEX_URL` for client-side Convex connection

### File Organization
```
├── app/                 # Next.js app router
│   ├── layout.tsx      # Root layout with providers
│   ├── page.tsx        # Home page
│   └── server/         # Server-side pages
├── components/         # React components
├── convex/            # Convex backend
│   ├── schema.ts      # Database schema
│   └── myFunctions.ts # Example functions
└── .cursor/rules/     # Cursor AI coding rules
```

## Important Notes

- Follow Convex coding guidelines from `.cursor/rules/convex_rules.mdc`
- Use the new Convex function syntax with validators
- Maintain parallel development workflow with npm-run-all
- TypeScript paths configured for `@/*` imports