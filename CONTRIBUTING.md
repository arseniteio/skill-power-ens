# Contributing

This guide explains how to propose a change to the Power ENS skills.

## Before you start

Open an issue before a large change, so that the approach is agreed before the work. Small fixes can go straight to a pull request.

Report security vulnerabilities privately to devqs@quersystem.com, not in an issue.

## Development setup

Install Claude Code or OpenCode. Load the repository as a local plugin, or clone it into the skills folder as the README describes.

## Checks

There is no automated test. Run the changed skill against the distributed lab of wazuh-lab-docker and check that the agent follows it from start to end.

## Code

- Code, identifiers, comments, commit messages and pull requests are in English.
- Follow the conventions already in the repository. Clear names, small functions and explicit error handling come before comments.
- Write a comment only for what the code cannot say: an invariant, an external constraint or a workaround with its reference.
- Tests live in their own files.
- A skill orchestrates the installer and the documented procedures. It never replaces them with its own steps.
- Never put host names, addresses, passwords or customer data of real environments in a skill or an example.

## Commits

Use [Conventional Commits](https://www.conventionalcommits.org/) with a scope, in one line of at most 72 characters, imperative and lowercase:

```text
fix(install): wait for the rules of the release on workers
```

Types: feat, fix, refactor, perf, test, docs, build, ci, chore, style and revert.

## Pull requests

- One logical change per pull request, on a branch named `feat/<topic>` or `fix/<topic>`.
- The checks above pass.
- New behaviour comes with tests, and a bug fix with a test that fails without it.

## License

The project is licensed under AGPL-3.0-only, and Arsenite also offers it under a commercial license. To keep both possible, Arsenite needs the rights to every contribution. Before your first pull request is merged, sign the Arsenite contributor license agreement. Ask for it at devqs@quersystem.com.
