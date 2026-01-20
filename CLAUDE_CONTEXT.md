# CLAUDE_CONTEXT.md

## Project
LivraOne Invoice (FlutterFlow-first app)

## Architecture Rules
- FlutterFlow is the UI builder. Code must be compatible with Custom Actions / Functions.
- No direct HTTP calls in UI. All calls go through ApiClient.
- Auth is MOCK now, REAL later via LivraOne Gateway (SSO).
- Never assume Firebase Auth.
- Never assume Supabase Auth.
- Always preserve orgId, roles, entitlements.

## Current Phase
PHASE 1 — Integration foundation (Auth + ApiClient + Mock)

## Forbidden
- Do not refactor UI.
- Do not introduce new state managers.
- Do not change file structure without request.

## Style
- Explicit
- Deterministic
- Minimal abstractions
