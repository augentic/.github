# Security Policy

This policy applies to every repository in the Augentic organisation that does
not carry a `SECURITY.md` of its own.

## Reporting a vulnerability

Please do not open a public issue for a security problem.

Report it privately through GitHub: open the affected repository's **Security**
tab and choose **Report a vulnerability**. That opens a private advisory that
only the maintainers can see, where the report, the fix, and the disclosure are
coordinated. If a repository has private reporting switched off, open a private
advisory on [`augentic/toolkit`](https://github.com/augentic/toolkit/security/advisories/new)
naming the affected repository, and the maintainers will move it.

A report is most useful with:

- the repository and the version or commit affected;
- what the vulnerability lets an attacker do;
- steps or a proof of concept that reproduce it;
- any mitigation you know of.

## What to expect

- An acknowledgement within five working days.
- An assessment of severity and the affected versions, shared with you in the
  advisory.
- A fix released as a patch of the current minor version, and a published
  advisory crediting you unless you ask otherwise.

Please give the maintainers reasonable time to release a fix before disclosing
the problem publicly; ninety days from the acknowledgement is the default.

## Supported versions

Augentic repositories are pre-1.0 and release from `main`. Security fixes land
on `main` and ship in the next release; the `release-<version>` branch of the
latest release receives a patch release where one exists. Older releases are
not patched.

## Supply chain

Every Rust repository audits its dependencies with
[`cargo vet`](https://mozilla.github.io/cargo-vet/) and checks advisories with
[`cargo deny`](https://embarkstudios.github.io/cargo-deny/) on every push and on
a schedule. A dependency advisory that affects a repository is handled as a
vulnerability in that repository.
