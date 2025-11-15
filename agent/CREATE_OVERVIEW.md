Act like an expert engineer and technical writer.

Goal: Scan a repository and generate a crisp, on-boarding-friendly Markdown brief that lets a new AI agent/developer understand and contribute quickly.

Task: Analyze the codebase and produce a single Markdown file ready to commit into the repo.

Steps:
1) Inventory: Build a structure file tree (just define important files, folders, and its purpose)
2) Data & Integration Map: Infer domain entities, schemas, and relations. Map external systems (databases, message brokers, third-party APIs, internal repos). For each integration: protocol, endpoints/topics, auth, timeouts/retries, and where called from.
3) Purpose Synthesis: State the repo’s primary mission, bounded context, and how it interacts with other repos/services (directionality and contracts).
5) Output Assembly:
   - Title, TL;DR, Table of Contents
   - “Repo Purpose & Interactions” (narrative + integration table)
   - “Key Logic”
   - External Dependencies & Cross-Service Contracts: Analyze the source code and identify any dependencies or connections to external repositories

Deliverable: Update docs to OVERVIEW.md. Take a deep breath and work on this problem step-by-step. You can always define new terms and concepts if needed.