# Hawaii Co-op Project

An open-source web platform that helps communities in Hawaii form and operate cooperative organizations — housing co-ops and agricultural co-ops.

## What It Does

Forming a co-op in Hawaii is hard. Hawaii has specific statutory requirements under HRS Chapter 421 (cooperatives) and Chapter 421H (housing cooperatives), and most organizers don't know where to start. This platform guides a group through every stage of the co-op lifecycle — from initial organizing, through legal formation, into ongoing operations — removing the expertise barrier that prevents most grassroots efforts from succeeding.

### Core Features (v1)

- **Formation Wizard** — Step-by-step guided flow from intent to a legally registered co-op, including DCCA filing instructions and Hawaii-specific document templates.
- **Membership Management** — Member registry, equity tracking, and admission workflows.
- **Governance** — Proposal and voting system aligned with co-op democratic principles, with quorum enforcement and an immutable audit trail.
- **Document Library** — Secure storage for governing documents with Hawaii-specific templates (Articles of Incorporation under HRS 421 and 421H, model bylaws, membership agreements).
- **Meeting Management** — Agenda builder, minute recording, and member notifications.
- **Advisor Access** — Lawyers, accountants, and USDA agents can be granted access across multiple co-ops.

## Hawaii-Specific Context

- Covers both housing and agricultural co-op types under Hawaii Revised Statutes
- Accommodates island-by-island variation (Oahu, Maui, Hawaii Island, Kauai, Molokai)
- Surfaces state programs available to agricultural co-ops (ALISH, Beginning Farmer programs)
- Designed with the *ohana* model of shared responsibility in mind

## Architecture

Multi-tenant SaaS application built on:
- **API layer** — REST or GraphQL
- **Database** — PostgreSQL with row-level security for tenant isolation
- **File storage** — S3-compatible object store for documents and templates
- **Auth** — JWT-based with optional Google OAuth

See [`docs/architecture.md`](docs/architecture.md) for the full system design.

## Documentation

- [`docs/overview.md`](docs/overview.md) — Product vision, target users, and success metrics
- [`docs/features.md`](docs/features.md) — Full v1 feature specification
- [`docs/architecture.md`](docs/architecture.md) — System architecture and data model

## Contributing

This project is in early stages. Contributions, feedback, and questions are welcome — open an issue to start a conversation.

## License

MIT — see [LICENSE](LICENSE)
