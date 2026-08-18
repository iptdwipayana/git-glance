# Git Glance

A small web app to search GitHub users and repositories — enter a `username` or
`username/repository` and get a quick look at it.

This is a personal learning project. The goal is to practice the stack below and
agent-assisted engineering, so clarity and correctness matter more than speed.

> [!WARNING]
> **This project is under active development.** Nothing is stable yet — the app is still
> being scaffolded, so the structure, APIs, and commands below are subject to change at
> any time.

## Tech stack

| Concern      | Choice                              |
| ------------ | ----------------------------------- |
| Language     | TypeScript (strict)                 |
| UI           | React                               |
| Routing      | TanStack Router                     |
| Server state | TanStack Query                      |
| Styling      | Tailwind CSS + shadcn/ui            |
| Testing      | Vitest                              |
| Lint/format  | Biome                               |

## Notes

Git Glance talks to the **unauthenticated** GitHub REST API, which is limited to 60
requests per hour (10 per minute for search). Searches are debounced and results are
cached generously to stay within that budget. No API token is required — or accepted.
