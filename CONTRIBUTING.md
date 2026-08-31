# Contributing

Thanks for taking an interest. This repository contains evidence-based case
studies about agent-driven software delivery.

## What makes a useful contribution

Corrections and improvements are welcome, especially when they make a claim
clearer or expose a limitation the article missed.

Please follow these rules:

- **Evidence before confidence.** A factual claim should point to something that
  can support it: code, history, a test result, a ledger record or an
  authoritative source.
- **Separate fact from interpretation.** Say what was observed, then label any
  conclusion drawn from it.
- **Keep the limits visible.** Do not turn “observed in this system” into “works
  everywhere.”
- **Do not invent missing detail.** If evidence is unavailable, say so.
- **Protect private information.** Remove internal identifiers, credentials,
  personal data and private business details.
- **Use natural professional English.** Prefer short, direct sentences without
  removing necessary technical terms.
- **Use Mermaid for diagrams.** Tables remain Markdown tables. Source code,
  configuration and terminal output remain fenced code blocks.

## Article structure

A deep dive should normally explain:

1. the problem and why it matters;
2. the system boundary and assumptions;
3. the mechanism or decision;
4. the evidence available;
5. failures and changes made in response;
6. known limits; and
7. what another team could reasonably reuse.

Not every article needs every heading, but the evidence and limitations should
never disappear for the sake of a cleaner story.

## Pull requests

Keep editorial changes focused and explain whether they alter wording, evidence
or the technical conclusion. If a change affects a reported number, include the
source and measurement window.

Contributions use the Developer Certificate of Origin. Sign off commits with
`git commit -s`.

## Licensing

Code contributions are licensed under Apache 2.0. Documentation contributions
are licensed under CC BY 4.0. See [LICENSE](LICENSE),
[LICENSE-docs.md](LICENSE-docs.md) and [DISCLAIMER.md](DISCLAIMER.md).
By opening a pull request, you agree that your contribution can be published
under those terms.
