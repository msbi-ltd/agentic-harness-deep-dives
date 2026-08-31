# Security policy

## What this repository contains

This is primarily a documentation repository. It contains case studies,
diagrams, data examples and small implementation samples. It does not run a
service or handle user credentials.

Some articles describe security-sensitive controls, including reviewer
isolation, protected workflows and commit-bound approval. A mistake in that
guidance could still lead someone to build an unsafe gate.

## Reporting

Use GitHub's **private vulnerability reporting** from the repository's Security
tab when that option is available and a finding could expose a real weakness in
a described control. If the option is unavailable, do not publish sensitive
details; open a minimal issue asking the maintainer for a private channel.

Use a normal public issue or pull request for ordinary corrections, broken links
or examples that are wrong but not security-sensitive.

## In scope

- reasoning that could cause a control to fail open;
- an example that grants more permission than the article claims;
- a workflow that allows the authoring identity to forge or bypass review;
- accidental publication of credentials, personal data or private system
  details; and
- a material mismatch between a security claim and the mechanism shown.

## Out of scope

- the general fact that examples are not production-ready;
- vulnerabilities in systems mentioned by the articles but not maintained in
  this repository;
- weaknesses that require a threat model the article explicitly excludes; and
- requests for security support or implementation consulting.
