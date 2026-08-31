# Agentic harness, deep dives

This repository contains longer case studies from a real autonomous coding
harness. Each article follows one problem through the controls, evidence,
failures and subsequent changes. You do not need to read the companion
repository first.

Shorter implementation recipes live in the
[agentic-harness-cookbook](https://github.com/msbi-ltd/agentic-harness-cookbook).
Use it when you want one reusable mechanism, such as a diff budget, commit-bound
review evidence, a three-outcome check, a mistake ledger or durable
controller/worker separation. The articles here link to those recipes where they
help explain the larger case.

The evidence comes from the source system's evaluation ledger, specifications
and review history. Identifiers are removed and internal text is paraphrased.
Numbers are reproduced from a dated shared snapshot, with the denominator and
limitations stated alongside them. When the records cannot support a claim, the
article says so.

## Articles

The articles build on one another, so this is the best reading order:

1. [Governing the double loop when the developer is an agent](articles/enforcing-the-double-loop-for-agents.md) looks at acceptance-test-driven development when a coding agent runs both loops. The agent can misread a requirement, weaken its own tests and edit the evidence used to judge its work. The article follows one case in detail, then compares it with 334 recorded work-item runs: 318 delivery runs and 16 separately recorded review-worker runs. Those 16 are a distinct ledger population, not a count of every external or Codex review.

2. [Evals for an agent loop, and the honesty problems in building them](articles/honest-evals-for-an-agent-loop.md) asks how we can tell whether the loop is actually improving. It starts with a metric that reported "100% clean" for six days while reading an empty ledger, then covers the safeguards we added in response.
   - [Building an honest eval ledger and dashboard](articles/building-an-honest-eval-ledger-and-dashboard.md) is the implementation companion, with record schemas, metric definitions, dashboard behaviour and the states a reported number can take.

3. [From conversational loop to durable controller](articles/architecture-evolution-from-conversational-loop-to-durable-controller.md) describes the architecture that followed. The existing controls still helped, but workflow liveness depended on long-lived conversations. The revised design separates worker heartbeat from workflow durability, capability provenance from display identity, and agent work from the protected control plane.

## Licence and project policies

- Documentation and Mermaid diagrams: [CC BY 4.0](LICENSE-docs.md)
- Code and executable samples: [Apache License 2.0](LICENSE)
- [Disclaimer](DISCLAIMER.md)
- [Contributing](CONTRIBUTING.md)
- [Security policy](SECURITY.md)
- [Copyright and trademark notice](NOTICE)
