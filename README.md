# @bffless/claude-skills

Claude Code plugin with platform skills for [BFFless](https://bffless.app) — a self-hosted static asset hosting platform with AI-powered pipelines.

These skills give Claude Code domain knowledge about BFFless features so it can help you build, deploy, and configure your projects.

## Skills Included

| Skill                 | Description                                              |
| --------------------- | -------------------------------------------------------- |
| **authorization**     | Global and project roles, API keys, permission model     |
| **bffless**           | Platform overview, key concepts, and feature summary     |
| **chat**              | AI chat widget/full-page, skills, streaming, persistence |
| **pipelines**         | Backend automation with handler chains and DB Records    |
| **proxy-rules**       | Forward requests to backend APIs, eliminate CORS         |
| **repository**        | Deployments, aliases, content browser, rollback          |
| **share-links**       | Token-based sharing for private deployments              |
| **traffic-splitting** | A/B testing, canary deployments, weighted routing        |
| **upload-artifact**   | GitHub Action for uploading builds to BFFless            |

## Install

### Via Claude Code Plugin Marketplace

```bash
# Add the marketplace
/plugin marketplace add bffless/claude-skills

# Install the plugin
/plugin install bffless
```

### Via CLI

```bash
claude plugin install bffless --scope user
```

### Local Development

If you're contributing or want to use a local copy:

```bash
git clone https://github.com/bffless/claude-skills.git
claude --plugin-dir ./claude-skills
```

## Usage

Once installed, Claude Code automatically uses these skills when you ask about BFFless features. For example:

- "Set up a proxy rule to forward /api requests to my backend"
- "Add AI chat to my site with streaming"
- "Configure traffic splitting for a canary deployment"
- "Set up the upload-artifact GitHub Action for my repo"

Skills are invoked by name with the `bffless` namespace:

```
/bffless:chat
/bffless:proxy-rules
/bffless:pipelines
```

## BFFless Pipeline Skills vs Claude Code Skills

This package contains **Claude Code plugin skills** — they teach Claude about the BFFless platform so it can assist you as a developer.

These are different from **BFFless pipeline skills**, which are markdown files you create in your own project's `.bffless/skills/` directory. Pipeline skills are deployed with your site and loaded by the AI chat handler at runtime to give your chatbot domain-specific knowledge.

|              | Claude Code Skills (this package)       | Pipeline Skills (your project)          |
| ------------ | --------------------------------------- | --------------------------------------- |
| **Purpose**  | Help Claude help _you_ build on BFFless | Give _your chatbot_ domain knowledge    |
| **Location** | Installed as Claude Code plugin         | `.bffless/skills/` in your project repo |
| **Used by**  | Claude Code (your dev tool)             | BFFless AI chat handler (your users)    |
| **Deployed** | npm install                             | `bffless/upload-artifact` GitHub Action |

## Documentation

- [BFFless Docs](https://docs.bffless.app)
- [Getting Started](https://docs.bffless.app/getting-started/quickstart)
- [Chat Feature](https://docs.bffless.app/features/chat/)
- [Pipelines](https://docs.bffless.app/features/pipelines/)

## Contributing

1. Fork this repo
2. Add or update skills in `skills/<skill-name>/SKILL.md`
3. Each skill needs YAML frontmatter with `name` and `description`
4. Open a PR — CI validates all skills automatically

## License

[O'Saasy](./LICENSE.md)
