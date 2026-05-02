---
layout: post
title: "Where the GitHub Copilot extension points break governance"
date: 2026-05-01
tags: [GitHub, GitHub Copilot, Security, Governance, MCP, VS Code, Copilot CLI, Skills, Plugins]
description: "A walkthrough of the governance gaps in the GitHub Copilot and extension surfaces: the Copilot CLI with its plugin marketplace, the Agent Package Manager (APM), gh skills, MCP servers across editors, and VS Code extensions through the Microsoft Marketplace and Open VSX."
---

A lot of the recent additions to the Copilot ecosystem add real value for individual developers, but they also expand the surface that an enterprise has to reason about. Most of these new entry points let a developer pull executable instructions, configuration, or full processes from any random repository on the internet, with very little or no central control. This post looks at the five places where I think the gap between "useful for one engineer" and "safe to run across a 5,000 person org" is widest right now.

## GitHub Copilot CLI plugin marketplace

The [Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-marketplace) lets you register a marketplace of plugins and install plugins from it. The on-ramp is one command:

```bash
copilot plugin marketplace add OWNER/REPO
copilot plugin install some-plugin@some-marketplace
```

A marketplace is just a GitHub repository with a `marketplace.json` file in `.github/plugin/`. There is no review, no signing, no central index. Two marketplaces (`copilot-plugins` and `awesome-copilot`) are registered by default, but any user can add any other repo, including a personal fork or a newly created account that copies a real plugin name with a small change.

XXX review with versioning of these things!

Plugins themselves are executable assets. They sit in the directory the marketplace points at, get pulled to the user's machine, and run in the user's shell context with whatever permissions the developer has. That is the same context as their git credentials, their cloud CLI sessions, and any local secrets in their environment.

What is missing for an enterprise:

- No setting on a GitHub Enterprise or organization to restrict which marketplaces a Copilot CLI user is allowed to add. The MCP private registry policy that exists for VS Code does not cover this. I'd want at least to restrict this to repos under the organizations control. 
- No way to require signed plugins, or plugins from a verified publisher.
- No audit trail on the GitHub side that tells you which plugins your developers installed and from where.
- Clear versioning out of the box. preferably with provenance signing build in. 

If you compare this to how npm or PyPI are usually handled in a regulated org (a private proxy, an allowlist, a vulnerability scanner in the pipeline), the Copilot CLI plugin story today is roughly where npm was around 2014.

## Agent Package Manager (APM)

[Microsoft APM](https://github.com/microsoft/apm) is a dependency manager for AI agent context. You declare an `apm.yml`, run `apm install`, and it pulls instructions, skills, prompts, agents, hooks, plugins, and MCP servers from any git host (GitHub, GitLab, Bitbucket, Azure DevOps, GitHub Enterprise) into every detected agent client on the machine.

After installing you point it to a local configuration file (can be in the current repo as well!) like this:

XXX: validate the repo part!

```yaml
dependencies:
  apm:
    - anthropics/skills/skills/frontend-design
    - github/awesome-copilot/plugins/context-engineering
    - microsoft/apm-sample-package#v1.0.0
  mcp:
    - name: io.github.github/github-mcp-server
      transport: http
```

APM does ship a governance story, and it is genuinely better thought through than most of the other entries in this post. There is `apm-policy.yml` with tighten-only inheritance from enterprise to org to repo, a published bypass contract, hidden-Unicode scanning on every install, lockfile integrity hashes, and an `apm audit --ci` mode that you can wire into branch protection.

The catch is that all of this governance is opt-in and lives outside of GitHub itself. A fresh install of `apm` on a developer laptop has no policy file. 
XXX check how this trickles down from the org. Is that automatic from tracing the repo upstream?

Until your security team writes one, publishes it, and gets it on every **single** machine, an `apm install` will happily resolve transitive dependencies from any reachable git host. And don’t worry: your CI/CD pipeline will do the same (if the tooling is already installed)!

XXX: check if the tooling can just be installed from inside a package config and be slipped in that way. 

XXX: check versioning and updates

Direct and transitive resolution are the parts I would worry about: a package you trusted six months ago can pull in a new dependency on its next release and there is no central registry or policy that says "this source/version is approved".

The MCP integration deserves its own line: APM will install a configured MCP server into every detected client (Copilot, Claude, Cursor, Codex, OpenCode, Gemini) in one command. XXX: check this mechanism! That is convenient and it is also a way to push an MCP server past whatever per-client policy exists, because the install happens at the filesystem level instead of through the editor's own registry flow. You can only hope that the tooling that runs them looks at a policy (hint: only a few handle this, and then still with loads of caveats and easy workarounds). 

## `gh skill` now in the GitHub CLI
Note: this is the **GitHub** CLI, not the GitHub **Copilot** CLI!

The [gh skill](https://github.blog/changelog/2026-04-16-manage-agent-skills-with-github-cli/) command lets you discover, install, manage, and publish [Agent Skills](https://agentskills.io) from any GitHub repository:

```bash
gh skill install github/awesome-copilot documentation-writer
gh skill install some-user/some-repo some-skill --pin v1.0.0
```

The supply-chain features here are better. Skills can be pinned to a tag (unsafe) or commit SHA, the install records the git tree SHA in the skill's frontmatter, `gh skill update` compares local SHAs against the remote, and `gh skill publish` will offer to enable [immutable releases](https://docs.github.com/repositories/releasing-projects-on-github/about-releases) so that even a repo admin cannot rewrite a published version.

XXX: check for audit options here!

The thing that is not there is an org-level allowlist. GitHub itself is unusually direct about this in the changelog:

> Skills are installed at your own discretion. They are not verified by GitHub and may contain prompt injections, hidden instructions, or malicious scripts. We strongly recommend inspecting the content of skills before installation.

So the tooling around a single skill is solid, the tooling around "which skills is my org allowed to use" is not. A developer can `gh skill install` from any public repo, and the agent host (Copilot, Claude Code, Cursor, Codex, Gemini, all the CLI options) will pick the skill up the next time it scans the directory. Skills are a first-class extension point for the agent's behavior, which means a malicious skill is closer to a custom system prompt than to a passive config file.

XXX: check Dependabot support for detecting and thus reviewing updates. 

## MCP servers across editors

This is the area where the situation has gotten more confusing rather than less, even though there has been real work on it. Back in March 2025 MCP exploded into the AI world: extensibility from anywhere into anything! Since then, a lot of servers and OSS repos turned about to be playing around with things. The hard part is that al those repos have since then been abandoned. Read the research on its stat here: XXX map EndorLabs review here. 

A short summary of where MCP server config lives today:

- VS Code: `.vscode/mcp.json`, user `settings.json`, or contributed by an installed VS Code extension. XXX: check how APM does this on the repo level
- Cursor, Windsurf, Codex, Claude Code, Gemini CLI, Copilot and other CLI’s: each has its own file in its own location, with its own schema variations.
- The remote Copilot agents (Cloud Agent, Spark, Spaces, Review Agent) each have their own configuration surface and can only be configured by a repo admin. 

GitHub did ship an [MCP private registry policy](https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-mcp-usage/configure-mcp-registry) for Copilot that lets an enterprise restrict which MCP servers Copilot users can connect to. Useful, and a real step forward, but at the time of writing it only applies inside Copilot in VS Code. The same Copilot identity used in JetBrains, Neovim, the CLI, Spark, Spaces, the Cloud Agent, or the Review Agent is not covered by that policy.

Two patterns make the policy easier to bypass than it looks:

1. Local stdio servers. The policy gates remote endpoints well, but a developer can drop a local MCP server config into their own user settings and the editor will start it without a registry check. XXX check this!
2. Extension-contributed servers. A VS Code extension can contribute MCP servers through its manifest. If an extension is allowed to install (and most orgs do not gate extensions tightly, see the next section), the MCP servers it contributes inherit the same trust as the extension itself. That sidesteps the registry policy entirely.

Even worse: clone the extension repo from github.com, build it, and just use the compiled VSIX file in VS Code!

So the practical state is: you can get a meaningful slice of governance for Copilot in VS Code if you set up the registry, and almost no governance for any of the other clients on the same laptop, all of which can reach the same internal systems. So we are not there yet, but at least a step in the right direction. 

## VS Code extensions and the registry split

The extension story is the oldest of the five, and it is the one that has changed shape most recently because of the Cursor and Windsurf-style forks. A few things that are worth being explicit about:

- The Microsoft Visual Studio Marketplace is closed to non-Microsoft products by its terms of use. Any VS Code fork (Cursor, Windsurf, VSCodium, Kiro, Antigravity, Positron) cannot legally use it.
- Those forks generally point at [Open VSX](https://open-vsx.org/), the Eclipse Foundation registry. Open VSX has a smaller catalog, less aggressive abuse handling historically, and a publish flow that is easier to ride.
- In 2026 the Eclipse Foundation launched the Open VSX Managed Registry as an SLA-backed paid tier, partly because the volume coming from AI editors started exceeding what the community instance could carry. XXX: check this!

For an org this means the threat model differs by editor, even when the developer thinks they are installing "the same extension". A name on the Microsoft Marketplace is not necessarily owned by the same publisher on Open VSX. Typosquats and copy-jobs of popular extensions show up regularly on both registries, and an extension is essentially arbitrary code in your editor process with access to your workspace files, your environment, and any tokens the editor holds.

The MCP angle ties back in here: an extension can contribute MCP servers, settings, and language model providers. So an extension that gets past your install policy can reintroduce all the things you tried to gate at the registry layer.

What helps in practice:

- The VS Code `extensions.allowed` and related policies, deployed through your endpoint management, so that only an allowlist of extensions can install at all.
- Mirroring Open VSX internally if you support fork editors, with a curated subset rather than a full passthrough.
- Treating new extension installs the same way you treat new npm dependencies: review, scan, and budget for the maintenance.

XXX: add a paragraph on endpoint protection here. And check Jesse Houwing’s blog for a relevant post!

## State of the plugin landscape for GitHub Copilot

If I line up the different surfaces by how much org-level governance is actually possible today:

| Surface | Org-level allowlist | Provenance / pinning | Notes |
| --- | --- | --- | --- |
| Copilot CLI plugin marketplace | None | None | Any GitHub repo can be a marketplace |
| APM | Yes, via `apm-policy.yml` | Lockfile + content hashes | Policy is opt-in, customer-owned |
| `gh skill` | None | Tag and SHA pinning | GitHub explicitly mentions verification |
| MCP servers | Limited (Copilot in VS Code only) | None standardized | Local stdio and extension-contributed servers bypass the policy |
| VS Code extensions | Yes, via VS Code policy | Marketplace + signature | Differs across forks and Open VSX |

The pattern across all of them is that the per-developer experience is great, the per-org enforcement is either absent or has to be assembled from policies that live in different places than the feature itself. None of these are unfixable, and APM in particular shows what the right shape looks like, but the gap between "shipped" and "safe to deploy at scale" is wider than the changelog posts suggest.

If you are responsible for any of this in a larger org, the short version of what I would do:

1. Decide which of these surfaces you want your developers to use at all. Default-allow is not a neutral choice.
2. For the ones you allow, pick the strongest available control today (VS Code extension policy, the Copilot MCP registry, an APM policy file) and ship it.
3. For the ones with no control today (CLI plugins, `gh skill`), at minimum log and review, and feed back to GitHub and Microsoft that this gap matters.
Overall, tighten your grip on endpoint protection and your firewall/proxy configurations.

The features themselves are not the problem. The missing layer is the same one every package ecosystem has had to grow eventually: a place for an org to say which sources it trusts, applied uniformly across every client that can pull from them.