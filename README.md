**English** | [中文](./README_zh.md)

# HitCC

## Complete reverse-engineering documentation of the full logic of Claude Code CLI v2.1.197

HitCC is not a source code repository. It is a documentation knowledge base for learning, analysis, and rewrite work.  
The goal of this project is not to reconstruct the original file tree, but to recover Claude Code CLI's core runtime logic, module boundaries, configuration system, and surrounding ecosystem as faithfully as possible, so it can serve as a stable reference for a runnable alternative or a high-similarity rewrite.

This project is not affiliated with Anthropic PBC. The repository does not contain Claude Code original source code, cracking content, or implementations intended to bypass product policies or permission mechanisms.

## Notes

To obtain the Claude Code CLI package analyzed by this repository:
```sh
npm pack @anthropic-ai/claude-code@2.1.197
```

The Python scripts under `recovery_tools/` can perform an initial round of cleanup on obfuscated or encrypted source code.

This repository is based on local analysis of Claude Code CLI v2.1.197 obtained through the method above and the extracted bundle from its native binary package. It does not rely on Anthropic PBC network services, including LLM inference services.

## Version 2.1.197 Update

The `docs/` knowledge base has been updated from the previous v2.1.84 baseline to Claude Code CLI v2.1.197. The current evidence model now reflects the newer distribution shape: a wrapper package, a platform native binary package, and a JavaScript bundle extracted from the native executable's `.bun` section.

This update refreshes the runtime entry chain, command surface, background and remote capabilities, Plugin / Skill, TUI, control plane, telemetry, abuse/risk/refusal, tool execution, and `contextLayers` topics. Several evidence-heavy pages have also been split into parent navigation pages and child topic pages so the evidence can be followed by runtime path.

It also adds a standalone topic page, [docs/standalone/abuse-control-and-telemetry.md](./docs/standalone/abuse-control-and-telemetry.md), for the client-side evidence around abuse/risk controls and telemetry. This page groups prompt markers, request identity, telemetry/logging channels, managed policy limits, execution gates, remote-control trust, gateway safeguards, and refusal/fallback handling into one cross-cutting view, while keeping private server-side enforcement outside the documented evidence boundary.

The documentation content in this repository may be quite extensive. The current statistics for the `docs/` directory (the following data may not be updated in real time):

```
TOTAL
  Files: 110
  Lines: 32013
  Chars: 961767
  Bytes: 1288945
```


## What This Repository Provides

- Structured reverse-engineering documentation based on the Claude Code CLI v2.1.197 local package and extracted bundle
- A topic-oriented evidence base organized around runtime, execution, ecosystem, rewrite, and appendix materials
- Boundary judgments, candidate architecture, and next evidence work for learning, analysis, and high-similarity rewrite efforts

## What This Repository Does Not Provide

- Reconstruction of the original source file structure
- Private server-side implementation details
- A guarantee of 1:1 behavioral reproduction
- A directly runnable CLI, SDK, or installation script

## Documentation Coverage

The current knowledge base is not organized by the original source tree. It is organized by runtime topics that can be reconstructed with stable confidence. The main knowledge base lives under `docs/docs/`, with standalone topics under `docs/standalone/`. At present, coverage can be understood by directory:

- `00-overview`
  - Scope boundaries, evidence sources, confidence terminology, and documentation maintenance conventions
- `01-runtime`
  - CLI entry, command tree, runtime mode dispatch, and Session / Transcript persistence and recovery
  - Input compilation pipeline, main Agent Loop, compact branch, model adapter, provider selection, and auth
  - Stream events, request telemetry/refusal, remote transport, and request build boundaries
  - Two built-in network tools: `web-search` and `web-fetch`
  - API lifecycle, startup telemetry, 1P event logging, nonessential traffic switches, control plane, and non-LLM network paths
  - Settings sources, paths, merging, caching, write-back, CLI injection schema, key consumers, and abuse/risk/refusal controls
- `02-execution`
  - Tool execution core, concurrent execution, Hook runtime, Permission / Sandbox / Approval
  - Instruction discovery, rules, prompt assembly, context layering, and request-level injection boundaries
  - Non-main-thread prompt paths, compat agent definitions, attachment lifecycle, context runtime, and tool-use context
- `03-ecosystem`
  - Resume, Fork, Sidechain, Subagent, and agent team
  - Remote persistence, bridge, bridge credentials, and Plan system
  - MCP, Skill, Plugin, TUI, Monitor / Advisor, and their runtime interaction boundaries
- `04-rewrite`
  - Candidate layering, directory skeleton, open-question tiers, blocking judgments, and next evidence work for rewrite engineering
- `05-appendix`
  - Unified terminology and evidence indexes such as the glossary, evidence map, topic maps, and key conclusion review
- `standalone`
  - Cross-directory topic notes outside the main navigation tree, used for evidence that spans runtime, execution, and ecosystem boundaries
  - [Abuse Control and Telemetry](./docs/standalone/abuse-control-and-telemetry.md), covering prompt markers, request identity, telemetry/logging, policy/gates, remote-control trust, gateway safeguards, and refusal/fallback handling within the locally provable client-side boundary

## Current Conclusions

For the current evidence boundary, rewrite feasibility, remaining unknowns, and next evidence work, see:

- [Scope, Evidence, and Conclusions](./docs/docs/00-overview/01-scope-and-evidence.md)
- [Rewrite Judgment, Open Question Tiers, and Next Evidence Work](./docs/docs/04-rewrite/02-open-questions-and-judgment.md)

## Maintenance Principles

This repository is maintained as a knowledge base rather than a single long-form document. The core principles are:

- `00-overview/00-index.md` is only the global entry point and does not duplicate main content
- Parent pages only provide navigation and boundary descriptions, not long-form mechanism analysis
- Child pages hold the actual content, evidence points, counter-evidence, and open questions
- Unknowns are centralized in the scope page and rewrite judgment page to avoid duplicate maintenance across multiple directory pages
- When new evidence appears, update the relevant topic page first, then backfill the glossary or evidence map

Detailed conventions are described in:

- [docs/docs/00-overview/02-document-style-and-structure-conventions.md](./docs/docs/00-overview/02-document-style-and-structure-conventions.md)

## License

Copyright (c) 2026 Hitmux contributors

This project is licensed under [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).

This means you may:

- Copy and redistribute the project content
- Modify, adapt, and republish it
- Use it in commercial or non-commercial scenarios

But you must:

- Preserve attribution to the original authors or source
- Include a link to the license
- Indicate whether changes were made

See [LICENSE](./LICENSE) in the repository root for the full license text.

## Acknowledgements

Thanks to the [Linux Do](https://linux.do) community for its support.

## Disclaimer

Please use this project only for learning, research, teaching, or legitimate engineering analysis.  
Any use of this project to infringe Anthropic PBC's lawful rights or to bypass product policies is unrelated to this project, and the risk is solely the user's responsibility.

The authors assume no legal responsibility for the accuracy or completeness of the documentation, or for any direct or indirect loss caused by using it, including but not limited to legal risk, technical failure, or data loss.
