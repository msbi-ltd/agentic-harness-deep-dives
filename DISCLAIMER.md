# Disclaimer

**Read this before applying anything described in this repository.**

## No warranty or liability

The documentation is published under CC BY 4.0. Code and executable samples are
published under Apache License 2.0. Both licences limit warranties and
liability. This file explains the practical meaning in plain English.

Everything is provided **as is**. Neither MSBI Ltd nor any contributor accepts
responsibility for loss, damage, outage, data loss, security incidents, cost or
other harm caused by using or relying on this material.

## These are case studies, not a supported product

The articles describe engineering work extracted from a private system and
scrubbed for publication. They are intended to explain decisions, failures,
controls and measurements. They are not a library, a managed service or a
drop-in implementation.

- The material has not received an independent security audit.
- There is no support, maintenance or compatibility commitment.
- Examples favour clarity over production hardening.
- GitHub, Codex, Claude and other platforms change. Behaviour verified when an
  article was written may change later.
- Reported numbers describe the stated system and measurement window. They do
  not predict results in another environment.

## Security and approval controls need particular care

Some articles discuss reviewer isolation, merge gates, credential boundaries
and autonomous delivery. A partly implemented control can be worse than no
control because it creates confidence without providing protection.

Before using one of these ideas:

- check it against your own threat model;
- read the stated limitations;
- test failure and unavailable states, not only the successful path; and
- obtain an independent review if failure would have a serious impact.

## Not professional advice

Nothing here is legal, security, compliance or professional consulting advice.
Using the repository does not create a client or advisory relationship. Seek
qualified advice where your circumstances require it.

## No vendor affiliation

This project is not affiliated with, endorsed by or sponsored by Anthropic,
OpenAI, GitHub or any other vendor named in the articles. Product names are used
only to identify the tools involved. See [NOTICE](NOTICE).

## Reporting a problem

Open a public issue for ordinary factual or technical corrections. For
sensitive findings, use GitHub's private vulnerability reporting when it is
available. See [SECURITY.md](SECURITY.md).
