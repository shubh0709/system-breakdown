# Boards & Real-Time Collaboration

Status: In Progress · 12 questions

## Question backlog

- [ ] **[JIRA-001](JIRA-001-real-time-kanban-updates.md)** — How will changes made by other users appear in real time on a Jira Kanban board? *(In Progress — SSE decision recorded)*
- [x] **[JIRA-002](JIRA-002-realtime-transport-selection.md)** — When should we choose polling, long polling, SSE, or WebSockets? *(Design discussion complete — SSE + HTTP chosen; implementation pending)*
- [ ] **[JIRA-003](JIRA-003-managing-concurrent-sse-connections.md)** — How do we manage thousands of concurrent SSE connections? *(Design discussion documented — implementation and capacity testing pending)*
- [ ] **[JIRA-004](JIRA-004-broadcasting-events-across-server-instances.md)** — How do we broadcast events across multiple server instances? *(Design discussion documented — implementation and scale testing pending)*
- [ ] **JIRA-005** — How do we maintain event ordering?
- [ ] **JIRA-006** — How does a reconnecting client recover missed events?
- [ ] **JIRA-007** — When do we replay events versus fetch a fresh snapshot?
- [ ] **JIRA-008** — How do we maintain consistency when a card moves between boards?
- [ ] **JIRA-009** — What happens when two users update the same card simultaneously?
- [ ] **JIRA-010** — How should clients handle duplicate and out-of-order events?
- [ ] **JIRA-011** — How do we efficiently load and render boards with thousands of cards?
- [ ] **JIRA-012** — What happens to an active real-time subscription when user permissions change?

## Learning workflow

When discussion starts, create a dedicated `JIRA-XXX-topic.md` page in this folder and link it from the backlog. Capture problem, requirements, initial approach, alternatives, trade-offs, failure scenarios, implementation, decisions, open questions, and related questions. Do not create question pages ahead of discussion.
