# Contributing

Thank you for contributing to `GilmanLab/docs`.

Use GitHub Discussions for architecture and documentation planning. Use GitHub
Issues for non-security bugs in the docs site or content. For vulnerabilities,
stop and follow [SECURITY.md](SECURITY.md) instead of using public channels.

## Pull Requests

1. Keep changes scoped to one documentation concern when practical.
2. Update the Docusaurus content and root pointers together when you move or
   rename major pages.
3. Make sure the site builds and type-checks before requesting review.
4. Describe reader-facing changes clearly in the pull request.

## Local Setup

Validate the repository:

```sh
moon ci --summary minimal
moon run docs:check
```

Start the local docs server:

```sh
moon run docs:start
```
