---
name: nebulab-shopify-theme-setup
description: >
  Use this skill any time the user wants to set up a Shopify theme project —
  whether they say "set up a theme", "initialize a Shopify theme", "create
  bin/setup and bin/dev", or anything that implies scaffolding Shopify theme
  tooling. Covers all setup scenarios: initializing a new theme in an empty
  directory, adding tooling to an existing theme, and configuring GitHub
  Actions workflows. Trigger even for casual phrasing like "get the theme ready"
  or "set up Shopify tooling here." Do NOT trigger for theme development tasks
  that aren't about initial project setup: deploying themes, editing Liquid
  templates, modifying theme settings, or running theme checks on existing code.
---

# Shopify Theme Setup

Sets up Shopify theme development tooling in the current directory.

## User questions

Everytime this skill needs to ask the user a question, use `AskUserQuestion` with a clear and concise question, and provide example answers if the expected input format isn't obvious. For yes/no questions, always clarify what a "yes" or "no" means in the context of the question to avoid confusion. For example, instead of asking "Do you want to proceed?", ask "Do you want to proceed with setting up a Shopify theme in the current directory? Answering yes will start the setup process, while answering no will exit the setup without making any changes."

## Prerequisites Check

This skill operates in your current folder. Ask the user if they want to proceed with setting up a Shopify theme in the current directory. If they say no, exit the skill gracefully. If they say yes, proceed with the steps below.

Then, verify these are installed before proceeding:

- **Node.js and npx** — npx is bundled with Node.js and is used to run Shopify CLI without a global install (`npx @shopify/cli@latest`). Any recent LTS version of Node.js will work.

## Locating the Skill Directory

Several steps below reference files bundled with this skill. Before starting, resolve the skill root once: check `.agents/skills/nebulab-shopify-theme-setup/` first (project scope); if it doesn't exist, use `~/.agents/skills/nebulab-shopify-theme-setup/` (user scope). All subsequent references to "the skill's `examples/`" or "the skill's `assets/`" use this resolved path.

## Setup Steps

### 1. Initialize Theme (if needed)

Only run this step if the current directory is empty. Before proceeding, ask the user whether they want to start from a custom GitHub repository (and get the URL), or some defined ones:
- [Horizon](https://github.com/Shopify/horizon)
- [Dawn](https://github.com/Shopify/dawn)
- [Skeleton](https://github.com/Shopify/skeleton)
Recommend Horizon: it is Shopify's production-ready reference theme and ships with a large set of well-structured components, making it an excellent starting point both for developers and for AI-assisted work. Then run `npx @shopify/cli@latest theme init . -u <repo-url>`, substituting the chosen URL. Skip this step entirely if the folder already contains a theme.

### 2. Setup Environment

Create a `.env` file in the project root containing `SHOPIFY_FLAG_STORE=<store_name>.myshopify.com`, or add the variable to it if the file already exists. This variable is picked up automatically by `shopify theme dev` to target the correct store, and is also used by several GitHub Actions workflows (e.g. theme preview and deployment). The store name need to be in these exact form: `<store_name>.myshopify.com` (eg. `example.myshopify.com`). Ask the user for the store name if not already set, and be sure to clarify the expected format (providing an example), and validate it.

If the env var is already set, check the format of the store name. If it doesn't match the expected format, try to correct it (e.g. if the user entered `example`, add `.myshopify.com` and update the env var). Ask the user to confirm the corrected store name before proceeding.

Make sure `.env` is listed in `.gitignore` to avoid accidentally committing store credentials.

### 3. Create bin scripts

Create a `bin/` directory if it doesn't exist, then create the following two scripts using the example files in the skill's `examples/bin/` directory as a reference (see *Locating the Skill Directory* above). Adapt the files to the project if needed, and always run `chmod +x` on each file after creating it.

- **`bin/dev`** — the single command developers run for daily work. It delegates to `bin/shopify theme dev`, forwarding any extra arguments. Keeping it as a thin wrapper means the underlying CLI invocation is centralised in `bin/shopify`, so swapping or upgrading the CLI only requires a change in one place.
- **`bin/shopify`** — invokes Shopify CLI via `npx --silent --yes -p @shopify/cli@latest shopify`, without requiring a global install. Using `npx` ensures every developer and CI environment always gets a compatible CLI version without any manual install step, and `--yes` suppresses interactive prompts so it works cleanly in automated contexts.

### 4. Setup GitHub Actions

For each workflow below, copy the corresponding file from the skill's `assets/github-workflows/` directory (see *Locating the Skill Directory* above) into `.github/workflows/`, creating the directory if needed. If a workflow file with the same name already exists, try to merge the new configuration with the existing one, preserving any custom steps or settings. If merging isn't possible, ask the user whether they want to replace the existing workflow with the new one. If they say no, skip that workflow and move on to the next one. If they say yes, replace the existing workflow file with the new one.

#### 4a. Theme Check (`theme-check.yml`)

Runs [`shopify/theme-check-action`](https://github.com/marketplace/actions/run-theme-check-on-shopify-theme) on every push to catch Liquid errors and theme best-practice violations. It checks the diff against `main` and posts results via the `GITHUB_TOKEN` secret, which is available automatically — no extra secret setup required. After copying the workflow file, initialise the theme check configuration by running `bin/shopify theme check --init` in the project root, but only if a `.theme-check.yml` file doesn't already exist. If the user needs more details on available configuration options, point them to the [action's Marketplace page](https://github.com/marketplace/actions/run-theme-check-on-shopify-theme).

#### 4b. Lighthouse CI (`lighthouse-ci.yml`)

Uses [`shopify/lighthouse-ci-action`](https://github.com/marketplace/actions/run-lighthouse-ci-on-shopify-theme) to run Google Lighthouse against a live theme preview on every push, enforcing minimum scores of 0.9 for both performance and accessibility. Requires some repository secrets: `SHOP_STORE` (the store URL, same value and format as `SHOPIFY_FLAG_STORE` in `.env`), `SHOP_CLIENT_ID` and `SHOP_CLIENT_SECRET` (credentials for a Shopify private app with `read_products` and `write_themes` permissions), and `LHCI_GITHUB_APP_TOKEN` (a token for the Lighthouse CI GitHub App to post results as commit statuses). For additional configuration options refer to the [action's Marketplace page](https://github.com/marketplace/actions/run-lighthouse-ci-on-shopify-theme).

#### 4c. PR Theme Preview (`pr-theme.yml`)

Uses [`ShopLab-Team/shoplab-pr-shopify-theme-preview`](https://github.com/marketplace/actions/shopify-pr-theme-preview) to deploy a temporary theme preview for every pull request and post the preview URL as a PR comment. When the PR is closed the preview theme is automatically cleaned up, and it can be re-deployed at any time by applying a `rebuild-theme` label. For security, the workflow only runs for PRs from the same repository and not from forks. Requires two repository secrets: `SHOP_STORE` (the store URL, same value and format as `SHOPIFY_FLAG_STORE` in `.env`) and `SHOPIFY_CLI_THEME_TOKEN` (a Shopify CLI theme token used to push and delete preview themes). For additional configuration options refer to the [action's Marketplace page](https://github.com/marketplace/actions/shopify-pr-theme-preview).

#### 4d. Secrets Setup

Ask the user if they need help setting up the required secrets for any of the above workflows. If they do, guide them through creating each secret in the GitHub repository settings, going to the GitHub repository, click on "Settings" > "Secrets and variables" > "Actions", then click "New repository secret". Provide also instructions on how to generate the necessary secret based on the following:
- **SHOP_STORE** — The same store URL used in the `.env` file (e.g. `example.myshopify.com`), with the `.myshopify.com` suffix.
- **SHOPIFY_CLI_THEME_TOKEN** — To generate a Shopify CLI theme token you have to add the Theme Access app to your store from the [Shopify App Store](https://apps.shopify.com/theme-access). Once installed, go to the app in your Shopify admin, click "Create password", fill all the required fields, and copy the generated token. This password provides read and write access to your themes using the Shopify CLI or Theme Kit CLI.
- **SHOP_CLIENT_ID** and **SHOP_CLIENT_SECRET** — Create a dev app in the Shopify following [this guide](https://github.com/marketplace/actions/run-lighthouse-ci-on-shopify-theme#dev-dashboard-app-recommended--required-for-apps-created-after-jan-2026), be sure to add `read_products` and `write_themes` accessscopes, then copy the generated client ID and client secret into the corresponding secrets.
- **LHCI_GITHUB_APP_TOKEN** — Add the Lighthouse CI GitHub App to your repository from the [GitHub Marketplace](https://github.com/apps/lighthouse-ci), then generate a token for it in the app's settings and copy it into the corresponding secret.

### 5. Final Confirmation

Print a success message confirming that the Shopify theme setup is complete and the project is ready for development. Recap the key steps that were taken, provide a quick overview of the created files and workflows, and encourage the user to start developing their theme using `bin/dev`.
