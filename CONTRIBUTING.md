# Contributing

Thanks for your interest in the Fiwano Cookbook 🙌

This repo is a step-by-step integration playbook plus a generated mirror of our API reference (runnable examples are on the way). A few notes before you contribute:

## What we welcome

- **Corrections** to the playbook, or an error spotted in the reference docs (we fix those at the source — see below).
- **Clarifications** in `README.md` or `PLAYBOOK.md`.
- **Bug fixes and new examples** once the `examples/` directory lands — in any language or for any use case.

## What not to edit

- **`reference/`** is **generated** from the canonical docs at [fiwano.com/documentation](https://fiwano.com/documentation). Don't edit files there — changes will be overwritten on the next sync. If you spot an error in the reference, open an issue or report it so we can fix it at the source.

## How to contribute

1. **Questions / how-to** → open a [Discussion](https://github.com/fiwano-com/fiwano-cookbook/discussions).
2. **A concrete bug or a request for a specific example** → open an [Issue](https://github.com/fiwano-com/fiwano-cookbook/issues).
3. **Code changes** → fork, create a branch, and open a pull request. Keep examples small, self-contained, and runnable.

## Ground rules

- Never commit secrets (API keys, tokens, webhook secrets). Use a `.env` file (git-ignored) and provide a `.env.example` instead.
- Match the style of the surrounding example.
- One example / one fix per pull request keeps review easy.
