# Changelog

All notable changes to the MediPulse project will be documented in this file.

## [Unreleased]
- Initialized project planning based on architectural documentation.
- Generated implementation plan for Next.js 14 App Router, Supabase, and Claude AI integration.
- Configured Next.js 14 project scaffold (`create-next-app`) within `medipulse` subdirectory.
- Integrated `shadcn-ui` components and Tailwind CSS with custom Medipulse Navy theme tokens.
- Scaffolded local Supabase migration tracking, porting initial schema definitions, enum types, and row level security policies.
- Implemented `@supabase/ssr` server/client instantiation and root routing middleware to enforce Doctor/Patient boundaries.
- Created `(auth)` login and signup pages and base Doctor/Patient dashboard layout scaffolds (`layout.tsx`).
