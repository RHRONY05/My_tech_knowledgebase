# 🧠 Personal Tech Knowledge Base (Obsidian Vault)

> **Personal Note:**  
> This repository is strictly for my personal use and continuous learning. Every time I learn a new technology or architecture pattern while building projects, I document it here in a centralized, evergreen format instead of letting it get trapped in project silos.

---

## 🎯 Purpose & Philosophy

Over months of building production software across different domains (E-commerce, Movie Booking, Project Management, AI applications), knowledge easily gets fragmented. When returning after breaks, relearning concepts from scratch causes unnecessary friction.

This repository turns past engineering experience into an **Obsidian-ready, permanent knowledge vault**:
1. **Topic-Based, NOT Project-Based:** Knowledge is organized by domain and tool (e.g., `backend/express/`, `database/mongodb/`, `devops/docker/`, `frontend/react/`), making it instantly discoverable regardless of which project it originated from.
2. **The 4-Question Standard:** Every note adheres to a battle-tested structure:
   - **The Core Problem:** Architectural and performance breakdown of why naive approaches fail.
   - **The Mental Model:** ASCII / Mermaid diagrams and real-world system analogies.
   - **Production Code Breakdown:** Battle-tested, copy-pasteable TypeScript/JavaScript implementations.
   - **Production Gotchas & Best Practices:** Real-world pitfalls, memory leaks, race conditions, and security hazards.
3. **Active Recall Revision System:** Every note starts with a diagnostic self-test checklist, indexed globally in [`00-active-recall/master-index.md`](./00-active-recall/master-index.md) for rapid interview and concept revision in under 60 seconds.

---

## 🗺️ Vault Structure

```text
My_tech_knowledgebase/
├── 00-active-recall/        # Interactive revision checklists, master index & scale lab roadmaps
├── backend/                 # Enterprise architecture (5 pillars), Express pipelines, Node, testing
│   ├── architecture/
│   ├── express/
│   ├── node/
│   └── testing/
├── database/                # Relational & NoSQL databases, caching, concurrency & migrations
│   ├── mongodb/
│   ├── postgresql/
│   └── redis/
├── devops/                  # Containerization, web servers, cloud deployments, security & CI/CD
│   ├── ci-cd/
│   ├── deployment/
│   ├── docker/
│   ├── nginx/
│   └── security/
├── frontend/                # Client-side routing, state management, form pipelines & linters
│   ├── react/
│   ├── redux/
│   └── tooling/
└── typescript/              # Static type contracts, compilation, generics & runtime schema safety
    ├── core/
    └── zod/
```

---

## ⚡ How to Use

- **Obsidian:** Open `My_tech_knowledgebase` as a local Obsidian Vault to explore the bidirectional graph view and cross-linked markdown files.
- **Active Recall Revision:** Head over to [`00-active-recall/master-index.md`](./00-active-recall/master-index.md), read the prompt questions out loud, and click into any note where the mental model feels rusty.
