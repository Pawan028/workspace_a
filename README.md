# CLN Power Mart — Platform

Discovery phase (started 2026-10-13). No code yet. Read in this order:

**Decisions so far:** [Team discussion and decision log](docs/discovery/05-discussion-log.md)

**Discovery**
1. [Project understanding](docs/discovery/01-project-understanding.md): deck digest, actors, scope, regulatory points, deck number checks
2. [Complexity and risk register](docs/discovery/02-complexity-and-risk-register.md)
3. [Client meeting agenda](docs/discovery/03-client-meeting-agenda.md): Part 0 holds the simple client questions; the rest is internal
4. [Delivery approach and releases R0–R3](docs/discovery/04-delivery-approach.md)

**Architecture**
1. [System design](docs/architecture/01-system-design.md): containers, modules, flows, AWS deployment, non-functional targets
2. [Financier integration](docs/architecture/02-financier-integration.md): multi-financier per product, offer engine, DO flow, waterfall, onboarding playbook, money map

**Engineering rulebook** (binding)
1. [Engineering principles](docs/engineering/01-engineering-principles.md)
2. [Coding standards](docs/engineering/02-coding-standards.md)
3. [Testing standards](docs/engineering/03-testing-standards.md)
4. [Security, privacy and compliance baseline](docs/engineering/04-security-privacy-compliance.md)
5. [Workflow, review, CI/CD, release](docs/engineering/05-workflow-git-release.md)
6. [Adversarial security rules](docs/engineering/06-adversarial-security.md): what attackers want, attack → control matrix, kill switches, verification program

**ADRs:** [Template](docs/adr/0000-template.md) · [0001 Stack (proposed)](docs/adr/0001-architecture-and-stack.md) · [0002 Financier plug-in](docs/adr/0002-financier-plug-in.md) · [0003 Hosted payment gateway](docs/adr/0003-payment-gateway-hosted-checkout.md)

Hard rules for everyone, including AI assistants: [CLAUDE.md](CLAUDE.md)

Team memory snapshot (project context, working agreements): [docs/memory](docs/memory/README.md)
