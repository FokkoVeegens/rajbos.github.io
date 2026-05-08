---
layout: post
title: "Where the GitHub Copilot extension points break governance"
date: 2026-05-01
tags: [GitHub, GitHub Copilot, Security, Governance, MCP, VS Code, Copilot CLI, Skills, Plugins]
description: "A walkthrough of the governance gaps in the GitHub Copilot and extension surfaces: the Copilot CLI with its plugin marketplace, the Agent Package Manager (APM), gh skills, MCP servers across editors, and VS Code extensions through the Microsoft Marketplace and Open VSX."
---

A lot of the recent additions to the GitHub Copilot ecosystem add real value for individual developers, yet they also expand the security surface that an enterprise has to reason about. Most of these new entry points let a developer pull executable instructions, configuration, or full processes from any random repository on the internet, with very little or no central control. This post looks at the five places where I think the gap between "useful for one engineer" and "safe to run across a 5,000 person org" is widest right now.

We'll look at these topis:
- GitHub Copilot CLI plugin marketplace
- GitHub Copilot CLI local extensions
- Agent Package Manager (APM)
- `gh skill` now in the GitHub CLI
- MCP servers across editors
- VS Code extensions and the different registries


![Samuel Regan Asante from Unsplash](/images/2026/20260501/samuel-regan-asante-STDn0DxY8os-unsplash.jpg)
##### Photo by <a href="https://unsplash.com/@reganography?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Samuel Regan-Asante</a> on <a href="https://unsplash.com/photos/a-sign-on-the-side-of-a-building-that-says-growing-concerns-STDn0DxY8os?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
      

## GitHub Copilot CLI plugin marketplace

The [Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-marketplace) lets you register a marketplace of plugins and install plugins from it. The on-ramp is one command:

```bash
copilot plugin marketplace add OWNER/REPO
copilot plugin install some-plugin@some-marketplace
```

A marketplace is just a GitHub repository with a `marketplace.json` file in `.github/plugin/`. There is no review, no signing, no central index. Two marketplaces (`copilot-plugins` and `awesome-copilot`) are registered by default, but any user can add any other repo, including a personal fork or a newly created account that copies a real plugin name with a small change.

Versioning is the next gap. The [CLI plugin reference](https://docs.github.com/en/copilot/reference/cli-plugin-reference) has a `version` field in `plugin.json`, but the `copilot plugin install` command has no syntax to pin to a version. You install `OWNER/REPO`, `OWNER/REPO:PATH`, a Git URL, a local path, or `plugin@marketplace`, and you get whatever HEAD of the source happens to be at that moment. `copilot plugin update NAME` pulls latest. There is no lockfile, no SHA pinning of the kind `gh skill` has, and no provenance attestation. A marketplace can change a plugin's contents without changing the `version` field, and the next `update` ships those changes to every developer who installed it.

Plugins themselves are executable assets.They sit in the directory the marketplace points at, get pulled to the user's machine, and run in the user's shell context with whatever permissions the developer has. That is the same context as their git credentials, their cloud CLI sessions, and any local secrets in their environment.

What is missing for an enterprise:

- No setting on a GitHub Enterprise or organization to restrict which marketplaces a Copilot CLI user is allowed to add. The MCP private registry policy that exists for VS Code does not cover this. I'd want at least to restrict this to repos under the organizations control. 
- No way to require signed plugins, or plugins from a verified publisher.
- No audit trail on the GitHub side that tells you which plugins your developers installed and from where.
- Clear versioning out of the box. preferably with provenance signing build in. 

If you compare this to how npm or PyPI are usually handled in a regulated org (a private proxy, an allowlist, a vulnerability scanner in the pipeline), the Copilot CLI plugin story today is roughly where npm was around 2014.

## GitHub Copilot CLI local extensions

Beyond the plugin marketplace there is a second, almost entirely undocumented extension surface baked into the Copilot CLI: the `.github/extensions/` directory. [A detailed reverse-engineering writeup by htek.dev](https://htek.dev/articles/github-copilot-cli-extensions-complete-guide/) extracted this from the Copilot SDK source, because there is essentially no public documentation for it. The architecture is that the CLI discovers any subdirectory containing an `extension.mjs` file, forks it as a child Node.js process, and communicates with it over JSON-RPC via stdio. The extension calls `joinSession()` and gets back a live session object that lets it register custom tools, intercept every agent lifecycle event, rewrite prompts, and make permission decisions.

The lifecycle hooks are the part that matters from a governance angle:

- `onSessionStart` — inject context into every session before the user's first message
- `onUserPromptSubmitted` — rewrite or augment the user's prompt before the agent sees it
- `onPreToolUse` — approve, deny, or modify the arguments of any tool call before it executes
- `onPostToolUse` — react after any tool completes, inject feedback the agent acts on
- `onErrorOccurred` — decide whether to retry, skip, or abort on failure

The `onPermissionRequest` handler in the `joinSession()` call replaces the standard user confirmation prompt for every tool execution. The SDK ships an `approveAll` import that does exactly what it sounds like: pass it and the extension silently approves every shell command, file write, and network call without showing the user a prompt.

The discovery paths are what make this a fleet-wide concern:

1. **Project-scoped**: `.github/extensions/` is committed to the repo. Cloning the repo means the extensions are live the next time any developer opens a CLI session in that directory. No `install` step. No opt-in. The moment the CLI opens the project, it forks whatever `.mjs` files are in that folder.
2. **User-scoped**: `~/.copilot/extensions/` applies to every repo on the developer's machine. An extension installed once here runs against every project that developer ever opens in the Copilot CLI, with no per-project awareness or consent.

Project extensions shadow user extensions on name collision, and the load order across extensions is not guaranteed — combined with a [known bug](https://github.com/github/copilot-cli/issues/2076) where multiple extensions registering hooks results in only the last-loaded extension's hooks actually firing, this creates silent behavior that is hard to audit even locally.

What is missing for an enterprise:

- No org-level allowlist. Any `.github/extensions/` directory in any repo a developer clones will have its extensions activated with no central gate.
- No signing or publisher verification. An extension is an arbitrary `.mjs` file on disk.
- No audit trail. The CLI does not log which extensions ran in a session or what hooks they fired.
- No policy to restrict which `onPermissionRequest` handlers can suppress user prompts. A malicious or compromised extension can run `approveAll` and the developer sees nothing different.

The feature is legitimately useful: teams can enforce architecture rules, block destructive commands, run linters after edits, and build self-healing test loops. But those same capabilities — prompt rewriting, permission suppression, tool argument modification — are exactly what a supply-chain attack would want, and right now there is no org-level control surface at all.

## Agent Package Manager (APM)

[Microsoft APM](https://github.com/microsoft/apm) is a dependency manager for AI agent context. You declare an `apm.yml`, run `apm install`, and it pulls instructions, skills, prompts, agents, hooks, plugins, and MCP servers from any git host (GitHub, GitLab, Bitbucket, Azure DevOps, GitHub Enterprise) into every detected agent client on the machine.

The manifest lives in the repo itself (`apm.yml` and `apm.lock.yaml` are committed alongside the code, like `package.json` and `package-lock.json`). It looks like this:

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

APM does ship a governance story, and it is the most thought-through one in this post. There is `apm-policy.yml` with tighten-only inheritance from enterprise to org to repo, a published bypass contract, hidden-Unicode scanning on every install, lockfile integrity hashes, and an `apm audit --ci` mode that you can wire into branch protection.

The catch is that all of this governance is opt-in and lives outside of GitHub itself. A fresh install of `apm` on a developer laptop has no policy file. Policy is also pull-only: the canonical org policy lives at `<org>/.github/apm-policy.yml` and the CLI fetches it on demand when a developer or CI job runs `apm install` or `apm audit --ci`. There is no push, no agent, and no central enrollment. The fetched policy is cached locally for an hour by default in `apm_modules/.policy-cache/`, and a `fetch_failure: warn|block` knob decides what happens when the org repo is unreachable. The default is `warn`, which means an offline laptop with an empty cache resolves with no policy at all. Repo-local `apm-policy.yml` files can `extends: org` and only **tighten** the parent rules, never relax them. See the [Policy Reference](https://microsoft.github.io/apm/enterprise/policy-reference/) and [Governance Guide](https://microsoft.github.io/apm/enterprise/governance-guide/) for the full mechanics.

Until your security team writes one, publishes it, and gets it picked up on every machine, an `apm install` will happily resolve transitive dependencies from any reachable git host. And don't worry: your CI/CD pipeline will do the same (if the tooling is already installed)!

There is also no auto-install. APM is purely a CLI; it has no editor extension that runs `apm install` when you open a repo in VS Code. The docs frame it explicitly as "same as `npm install` after cloning a Node project", which means the install step relies on the developer running it (or a devcontainer `postCreateCommand`, or a CI job). The flip side is that the **deployed** files (under `.github/`, `.claude/`, `.cursor/`, `.gemini/`) are recommended to be committed, so a teammate who clones the repo gets the agent context immediately, before they ever run `apm install`. That is convenient and it also means the agent is reading APM-deployed content the moment the editor opens the repo, regardless of whether the local CLI was ever invoked.

APM packages can declare `scripts` (think npm scripts), and the policy reference exposes `manifest.scripts: allow|deny` precisely because of this risk. Default is `allow`. So an attacker who lands a package in your dependency tree can also land scripts, unless your org policy denies them outright.

Versioning is fine on the manifest side: dependencies pin with `#tag` or `#sha`, the lockfile records resolved commit SHAs and content hashes, and the org policy can `require` specific versions with a `require_resolution` of `project-wins`, `policy-wins`, or `block`. Updates happen on `apm install --update`, not implicitly. Direct and transitive resolution stay the parts I would worry about: a package you trusted six months ago can pull in a new dependency on its next release, and unless your org policy has a tight `dependencies.allow` pattern, the new source slips through.

The MCP integration is worth a separate paragraph. `apm install --mcp NAME` adds an entry under `dependencies.mcp` in `apm.yml` and writes the resolved server config straight into the native config file of every detected client (Copilot, Claude, Cursor, Codex, OpenCode, Gemini) on the filesystem, bypassing each client's own registry or policy layer. The full mechanism is documented in the [APM MCP Servers guide](https://microsoft.github.io/apm/guides/mcp-servers/). Convenient for a developer; also a clean way around whatever per-client policy exists. You are then relying on the runtime side of those clients to apply policy, and only a few of them do, with workarounds.

## `gh skill` now in the GitHub CLI
Note: this is the **GitHub** CLI, not the GitHub **Copilot** CLI!

The [gh skill](https://github.blog/changelog/2026-04-16-manage-agent-skills-with-github-cli/) command lets you discover, install, manage, and publish [Agent Skills](https://agentskills.io) from any GitHub repository:

```bash
gh skill install github/awesome-copilot documentation-writer
gh skill install some-user/some-repo some-skill --pin v1.0.0
```

The supply-chain features here are better. Skills can be pinned to a tag (unsafe) or commit SHA, the install records the git tree SHA in the skill's frontmatter, `gh skill update` compares local SHAs against the remote, and `gh skill publish` will offer to enable [immutable releases](https://docs.github.com/repositories/releasing-projects-on-github/about-releases) so that even a repo admin cannot rewrite a published version.

Audit tooling is thin. The [`gh skill` manual](https://cli.github.com/manual/gh_skill) lists `install`, `preview`, `publish`, `search`, and `update` as the only subcommands. There is a `gh skill preview` to inspect a skill's content before installing, and `gh skill update` uses the stored tree SHA to detect drift, but there is no `gh skill audit` and no org-side audit log of what your developers installed. If you want to know which skills landed on a developer's laptop, you have to grep the agent host directories yourself.

The thing that is not there is an org-level allowlist.GitHub itself is unusually direct about this in the changelog:

> Skills are installed at your own discretion. They are not verified by GitHub and may contain prompt injections, hidden instructions, or malicious scripts. We strongly recommend inspecting the content of skills before installation.

So the tooling around a single skill is solid, the tooling around "which skills is my org allowed to use" is not. A developer can `gh skill install` from any public repo, and the agent host (Copilot, Claude Code, Cursor, Codex, Gemini, all the CLI options) will pick the skill up the next time it scans the directory. Skills are a first-class extension point for the agent's behavior, which means a malicious skill is closer to a custom system prompt than to a passive config file.

Dependabot does not help here either: agent skills, MCP servers, APM packages, and Copilot CLI plugins are not on the [list of ecosystems Dependabot supports](https://docs.github.com/en/code-security/dependabot/ecosystems-supported-by-dependabot/supported-ecosystems-and-repositories). That means no automatic update PRs, no security advisories wired in, and no scheduled drift detection across these surfaces. You would have to build that yourself.

## MCP servers across editors

This is the area where the situation has gotten more confusing rather than less, even though there has been real work on it. Back in March 2025 MCP exploded into the AI world: extensibility from anywhere into anything! Since then, a lot of servers and OSS repos turned out to be playing around with things. The hard part is that a large share of those repos have since been abandoned. Endor Labs covered this in its [State of Dependency Management 2025 report](https://www.endorlabs.com/learn/state-of-dependency-management-2025) (summary on the [Endor Labs press release](https://www.prnewswire.com/news-releases/endor-labs-launches-2025-state-of-dependency-management-report-finds-80-of-ai-suggested-dependencies-contain-risks-302603438.html)): more than 10,000 MCP servers were created in less than a year, 75% of them by individual developers rather than organizations, around 40% have no license at all, and 82% touch sensitive APIs. Maintenance signals on the long tail are weak, which means the same servers your developers happily installed last year may already be effectively orphaned.

A short summary of where MCP server config lives today:

- VS Code: `.vscode/mcp.json` (workspace), the user-profile `mcp.json` opened via `MCP: Open User Configuration`, or contributed by an installed VS Code extension. The full schema is in the [VS Code MCP servers docs](https://code.visualstudio.com/docs/copilot/customization/mcp-servers). APM sits on top of this: it stores MCP servers in the repo's `apm.yml`, then writes them into the same `.vscode/mcp.json` file the editor reads. By default `apm install` does not overwrite locally-authored entries (that needs `--force`, per the [CLI reference](https://microsoft.github.io/apm/reference/cli-commands/)), so the file you end up with is APM's set **plus** anything that was already there. If a developer thinks "I'm only running the APM-managed servers", they are wrong: they are running APM-managed plus whatever they (or another tool) wrote into `mcp.json` previously.
- Cursor, Windsurf, Codex, Claude Code, Gemini CLI, Copilot and other CLI’s: each has its own file in its own location, with its own schema variations.
- The remote Copilot agents (Cloud Agent, Spark, Spaces, Review Agent) each have their own configuration surface and can only be configured by a repo admin. 

GitHub did ship an [MCP private registry policy](https://docs.github.com/en/copilot/how-tos/administer-copilot/manage-mcp-usage/configure-mcp-registry) for Copilot that lets an enterprise restrict which MCP servers Copilot users can connect to. Useful, and a real step forward, but at the time of writing it only applies inside Copilot in VS Code. The same Copilot identity used in JetBrains, Neovim, the CLI, Spark, Spaces, the Cloud Agent, or the Review Agent is not covered by that policy.

Two patterns make the policy easier to bypass than it looks:

1. Local stdio servers. The [VS Code MCP docs](https://code.visualstudio.com/docs/copilot/customization/mcp-servers) describe three config paths: the gallery flow (which the Copilot private registry can gate), the workspace `.vscode/mcp.json`, and the user-profile `mcp.json`. The registry policy applies to the gallery flow. A developer who edits either JSON file directly gets a one-time "trust this server" prompt and the server starts. There is a separate VS Code device-management policy that can disable MCP entirely, but it is on/off, not allowlist-aware. See [extension runtime security](https://code.visualstudio.com/docs/configure/extensions/extension-runtime-security) for the surrounding policy surface.
2. Extension-contributed servers. A VS Code extension can contribute MCP servers through its manifest. If an extension is allowed to install (and most orgs do not gate extensions tightly, see the next section), the MCP servers it contributes inherit the same trust as the extension itself. That sidesteps the registry policy entirely.

Even worse: clone the extension repo from github.com, build it, and just use the compiled VSIX file in VS Code!

So the practical state is: you can get a meaningful slice of governance for Copilot in VS Code if you set up the registry, and almost no governance for any of the other clients on the same laptop, all of which can reach the same internal systems. So we are not there yet, but at least a step in the right direction. 

## VS Code extensions and the registry split

The extension story is the oldest of the five, and it is the one that has changed shape most recently because of the Cursor and Windsurf-style forks. A few things to be explicit about:

- The Microsoft Visual Studio Marketplace is closed to non-Microsoft products by its terms of use. Any VS Code fork (Cursor, Windsurf, VSCodium, Kiro, Antigravity, Positron) cannot legally use it.
- Those forks generally point at [Open VSX](https://open-vsx.org/), the Eclipse Foundation registry. Open VSX has a smaller catalog, less aggressive abuse handling historically, and a publish flow that is easier to ride.
- On April 21, 2026 the Eclipse Foundation [launched the Open VSX Managed Registry](https://newsroom.eclipse.org/news/announcements/eclipse-foundation-launches-open-vsx-managed-registry-0) as an SLA-backed paid tier (99.95% uptime, defined support tiers), with AWS, Google, and Cursor as initial adopters. The launch numbers paint the scale: 300M+ downloads per month, 200M+ daily requests at peak, 12,000+ extensions, 8,000+ publishers. The community instance was being asked to do the job of always-on critical infrastructure, and the AI editors are most of the reason.

For an org this means the threat model differs by editor, even when the developer thinks they are installing "the same extension". A name on the Microsoft Marketplace is not necessarily owned by the same publisher on Open VSX. Typosquats and copy-jobs of popular extensions show up regularly on both registries, and an extension is essentially arbitrary code in your editor process with access to your workspace files, your environment, and any tokens the editor holds.

The MCP angle ties back in here: an extension can contribute MCP servers, settings, and language model providers. So an extension that gets past your install policy can reintroduce all the things you tried to gate at the registry layer.

What helps in practice:

- The VS Code `extensions.allowed` and related policies, deployed through your endpoint management, so that only an allowlist of extensions can install at all.
- Mirroring Open VSX internally if you support fork editors, with a curated subset rather than a full passthrough.
- Treating new extension installs the same way you treat new npm dependencies: review, scan, and budget for the maintenance.

Endpoint protection is the layer that catches what the registries miss. Even the official VS Code documentation on [extension runtime security](https://code.visualstudio.com/docs/configure/extensions/extension-runtime-security) is direct that an extension runs with the user's full permissions: it can read and write any file the editor can, spawn processes, and make network calls. The Marketplace does scan packages and verify signatures (see the Microsoft post on [security and trust in the Visual Studio Marketplace](https://developer.microsoft.com/blog/security-and-trust-in-visual-studio-marketplace)), but malicious extensions and credential-stealing supply chain incidents keep landing (see the [Wiz writeup on supply chain risk in VS Code extension marketplaces](https://www.wiz.io/blog/supply-chain-risk-in-vscode-extension-marketplaces) and Check Point's [report on 45,000+ downloads of malicious extensions](https://blog.checkpoint.com/securing-the-cloud/malicious-vscode-extensions-with-more-than-45k-downloads-steal-pii-and-enable-backdoors/)). For an org that means the controls have to live below the editor: managed device policy that blocks unsigned binaries, EDR that watches the editor's process tree the same way it watches a browser, outbound DNS and TLS inspection that can flag the unusual call patterns an extension makes, and a workstation lifecycle that assumes a compromised editor is one of the realistic incidents you respond to. Third-party scanners like [ExtensionTotal](https://extensiontotal.com) can give you a per-extension risk score before you allow it, but treat them as an addition to your endpoint stack rather than a substitute.

## New: VS Code enterprise policy updates

VS Code shipped a notable batch of new enterprise policies around late April 2026 (documented at [code.visualstudio.com/docs/enterprise/policies](https://code.visualstudio.com/docs/enterprise/policies)) that start closing some of the gaps described above. The most relevant additions:

**MCP server control** — `ChatMCP` (`chat.mcp.access`) lets an admin disable access to all installed MCP servers via device policy. This is a blunter but more reliable control than the Copilot private registry alone, because it applies regardless of how the server was registered.

**Network filtering for agent tools** — `ChatAgentNetworkFilter`, `ChatAgentAllowedNetworkDomains`, and `ChatAgentDeniedNetworkDomains` let you restrict which hosts an agent's fetch tool and integrated browser can reach. Combined with `ChatAgentSandboxEnabled`, which runs terminal commands in a sandboxed environment, this starts to limit the blast radius of a compromised or malicious tool.

**Agent tool approval** — `ChatToolsAutoApprove` lets you lock down the "YOLO mode" (`chat.tools.global.autoApprove`) at the org level so individual developers cannot enable it, and `ChatToolsEligibleForAutoApproval` lets you force specific tools to always require manual confirmation.

**Account-gated AI access** — `ChatApprovedAccountOrganizations` blocks all AI features until the user signs into a GitHub account belonging to an approved organization. Useful for contractors, BYOD, and split environments where you need to tie the Copilot seat to an identity your org controls before anything runs.

**Linux policy support** — VS Code 1.106 added a `/etc/vscode/policy.json` file for Linux devices, meaning the same policies you deploy on Windows and macOS via ADMX or `.mobileconfig` can now cover Linux developer workstations through your existing config management tooling (Ansible, Puppet, Chef, Salt).

**Policy diagnostics** — A new `Developer: Policy Diagnostics` command generates a Markdown report of which policies are active, what values are in effect, and whether the Account Policy Gate is satisfied or blocked. Useful when you need to prove to an auditor — or a confused developer — exactly what the machine is enforcing.

These additions meaningfully strengthen the VS Code row in the summary table below. The gaps at the Copilot CLI, `gh skill`, and cross-editor MCP layers remain open.

## State of the plugin governance for GitHub Copilot

If I line up the different surfaces by how much org-level governance is actually possible today:

| Surface | Org-level allowlist | Provenance / pinning | Notes |
| --- | --- | --- | --- |
| Copilot CLI plugin marketplace | None | None | Any GitHub repo can be a marketplace |
| Copilot CLI local extensions | None | None | Committed to repo; active on clone with no install step |
| APM | Yes, via `apm-policy.yml` | Lockfile + content hashes | Policy is opt-in, customer-owned |
| `gh skill` | None | Tag and SHA pinning | GitHub explicitly mentions verification |
| MCP servers | Limited (Copilot in VS Code only) | None standardized | Local stdio and extension-contributed servers bypass the policy |
| VS Code extensions | Yes, via VS Code policy | Marketplace + signature | Differs across forks and Open VSX |

The pattern across all of them is that the per-developer experience is great, the per-org enforcement is either absent or has to be assembled from policies that live in different places than the feature itself. None of these are unfixable, and APM in particular shows what the right shape looks like, but the gap between "shipped" and "safe to deploy at scale" is wider than the changelog posts suggest.

If you are responsible for any of this in a larger org, the short version of what I would do:

1. Decide which of these surfaces you want your developers to use at all. Default-allow is a choice that has consequences, not a neutral starting point.
2. For the ones you allow, pick the strongest available control today (VS Code extension policy, the Copilot MCP registry, an APM policy file) and ship it.
3. For the ones with no control today (CLI plugins, `gh skill`), at minimum log and review, and feed back to GitHub and Microsoft that this gap matters.
Overall, tighten your grip on endpoint protection and your firewall/proxy configurations.

The features themselves are fine. The missing layer is the one every package ecosystem has had to grow eventually: a place for an org to say which sources it trusts, applied uniformly across every client that can pull from them.