# Licensing, in plain language

This page is a summary. The [LICENSE](LICENSE) file is the governing text; if
this summary and the LICENSE disagree, the LICENSE wins.

gjxdiff is proprietary software, distributed as a binary. It is not open
source and the source code is not available.

## Free use

- You are an **individual**: free, for anything, including your own scripts,
  cron jobs and CI.
- You work at an **organization with fewer than 100 people** (employees plus
  contractors, affiliates counted together): free, for anything, including
  the organization's automation and CI.
- You work at a **larger organization**: running gjxdiff by hand for your own
  task is free. Automated use (CI pipelines, servers, scheduled jobs,
  production processes) requires a commercial license.

All free use requires attribution:

> gjxdiff by Tibor Kovacs (Kotysoft)

A non-prominent placement is enough: an about, info or credits section, or
the accompanying documentation. For personal command-line use there is
nothing to attach the credit to, so there is nothing to do.

## Redistribution and embedding

- Passing the unmodified binary along free of charge, with all notices
  intact, is allowed.
- Shipping gjxdiff inside a **free, non-monetized** application is allowed,
  with attribution.
- Shipping it inside a **commercial** product, or any product that involves
  ads, paid tiers or data monetization, requires a commercial license.

## Not allowed

- Modifying the binary or building derivative works from it.
- Reverse engineering, except where the law allows it regardless of the
  license.
- Removing copyright or attribution notices.
- Selling or renting the binary on its own.
- Dressing up automated use as interactive use, or misstating organization
  size.

## Commercial licenses

For automated use in larger organizations and for embedding in commercial
products: support@giantjson.com

## Other terms

- No promised support, maintenance or updates. What ships, ships.
- Feedback and bug reports you send may be used freely by the author.
- A first license breach can be cured within 32 days of notice.
- Each release is governed by the license text shipped with it. A later
  license version never rewrites the terms for a copy you already have.

## Third-party components

The binary statically links a number of open-source components (all
permissively licensed; the full list with copyright notices is in the
THIRD-PARTY-NOTICES.md file shipped in the release archive, and
`gjxdiff --credits` prints the same list). Their licenses grant broader
rights for those components than the gjxdiff license does; LICENSE
section 6 says so explicitly.

## Warranty

None. The software is provided as is. See LICENSE sections 9 and 11.
