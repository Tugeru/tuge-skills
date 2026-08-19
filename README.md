# tuge-skills

A collection of agentic skills for the [Oh My Pi](https://github.com/oh-my-pi) (OMP) coding harness — drafted and forked workflows that coordinate vertical-slice implementation through GitHub trackers.

<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li><a href="#about-the-project">About The Project</a></li>
    <li><a href="#built-with">Built With</a></li>
    <li><a href="#getting-started">Getting Started</a></li>
    <li><a href="#usage">Usage</a></li>
    <li><a href="#skills">Skills</a></li>
    <li><a href="#skill-structure">Skill Structure</a></li>
    <li><a href="#contributing">Contributing</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
  </ol>
</details>

<!-- ABOUT THE PROJECT -->
## About The Project

Each top-level directory in this repository is an installable OMP skill. A skill defines agent behavior through a YAML-frontmatter `SKILL.md` file and, optionally, supporting reference documents (`.md` files loaded on demand by that skill).

Skills here handle three concerns:

- **Orchestration** — dispatching vertical slices under a GitHub parent issue, supervising workers through Herdr or Paseo worktrees, and serializing merges.
- **Planning & review** — stress-testing plans and specifications before implementation.
- **Handoff** — passing active work between models, providers, or agent sessions without losing state.

<p align="right"><a href="#readme-top">back to top</a></p>

### Built With

Skills execute against the following tooling chain:

- [Oh My Pi](https://github.com/oh-my-pi) — the agentic coding harness that loads and invokes skills
- [Herdr](https://github.com/herdr) — worktree lifecycle and agent management
- [Paseo](https://github.com/getpaseo/paseo) — daemon and CLI for launching and supervising coding agents in worktree workspaces
- [Orca CLI](https://github.com/orca) — terminal, worktree, and browser control
- [GitHub CLI](https://cli.github.com/) — issue, PR, and GraphQL operations

<p align="right"><a href="#readme-top">back to top</a></p>

<!-- GETTING STARTED -->
## Getting Started

### Prerequisites

- Oh My Pi (OMP) installed and configured
- `herdr` binary on `PATH`
- `gh` authenticated with repository access
- `git` with a clone of the target repository

### Installation

Clone this repository and place or symlink skill directories into your OMP skills path:

```sh
git clone https://github.com/tuge/tuge-skills.git
cd tuge-skills
```

OMP auto-discovers skills from directories containing a `SKILL.md`. No build step is required.

<p align="right"><a href="#readme-top">back to top</a></p>

<!-- USAGE -->
## Usage

Invoke a skill from within an OMP session by its frontmatter `name`:

```sh
/<skill-name> <arguments>
```

For example, to ship a single vertical slice tracked in GitHub issue #42:

```sh
/ship-slice 42
```

Arguments follow each skill's `argument-hint`. Some skills (e.g. `orchestrate-slices`, `model-relay`) carry `disable-model-invocation: true` in their frontmatter, meaning they coordinate without spawning a coding model directly.

<p align="right"><a href="#readme-top">back to top</a></p>

<!-- SKILLS -->
## Skills

| Skill | Directory | Purpose |
| --- | --- | --- |
| `orchestrate-slices` | [orchestrate-slices/](orchestrate-slices/) | Coordinate all vertical slices under a GitHub parent issue with a hard two-worktree ceiling, supervising workers and serializing merges. |
| `paseo-orchestrate-slice` | [paseo-orchestrate-slice/](paseo-orchestrate-slice/) | Same coordination with Paseo: dispatch vertical slices into Paseo worktree workspaces under a hard two-worktree ceiling, supervise workers, serialize merges. |
| `paseo-ship-slice` | [paseo-ship-slice/](paseo-ship-slice/) | Ship one vertical slice from ticket to merge-ready PR inside a Paseo worktree workspace. |
| `ship-slice` | [ship-slice/](ship-slice/) | Ship one vertical slice from ticket to merge-ready PR — implement, verify, review loop, push; orchestrated or standalone mode. |
| `slice-relay` | [slice-relay/](slice-relay/) | Relay a parent GitHub tracker through its vertical slices, passing the baton to sibling worktrees. |
| `faultline` | [faultline/](faultline/) | Stress-test a plan, design, or specification by identifying material gaps that could change implementation or scope. |
| `faultline-with-docs` | [faultline-with-docs/](faultline-with-docs/) | Faultline combined with domain modeling — gap discovery plus `CONTEXT.md` and ADR recording. |
| `model-relay` | [model-relay/](model-relay/) | Resume active work after switching models or providers by reconstructing state from artifacts. |
| `to-tickets` | [to-tickets/](to-tickets/) | Break a plan, spec, or conversation into tracer-bullet vertical-slice tickets with blocking edges published to a tracker. |
| `laymans-grilling` | [laymans-grilling/](laymans-grilling/) | Grill the user relentlessly about implementation and spec gaps in a plan or design — waves of 5 questions. |

<p align="right"><a href="#readme-top">back to top</a></p>

<!-- SKILL STRUCTURE -->
## Skill Structure

Each skill directory follows a consistent layout:

```text
skill-name/
├── SKILL.md              # YAML frontmatter + main instructions
├── REFERENCE.md          # (optional) reference data loaded on demand
├── SURVEY.md             # (optional) read-only survey/procedure
├── SUPERVISE.md          # (optional) supervision & merge gate
├── LAUNCH.md             # (optional) baton/worktree launch
└── WORKTREES.md          # (optional) worktree lifecycle reference
```

The `SKILL.md` frontmatter declares:

| Field | Description |
| --- | --- |
| `name` | The slash command used to invoke the skill. |
| `description` | One-line summary shown in skill listings. |
| `argument-hint` | Placeholder text guiding positional arguments. |
| `disable-model-invocation` | When `true`, the skill coordinates without spawning a coding model. |

Supporting `.md` files are loaded by the skill at specific steps — never by the user directly. Each reference file is self-contained and documents exactly one concern (survey queries, merge gates, worktree lifecycle, etc.).

<p align="right"><a href="#readme-top">back to top</a></p>

<!-- CONTRIBUTING -->
## Contributing

Contributions are welcome. Fork the project and open a pull request to add, refine, or split a skill.

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feat/new-skill`)
3. Write a `SKILL.md` with frontmatter and step-by-step instructions
4. Commit your Changes (`git commit -m 'feat: add new-skill'`)
5. Push to the Branch (`git push origin feature/new-skill`)
6. Open a Pull Request

When authoring skills, prefer reusing existing conventions over creating new ones. Each step should end with a `Done when:` criterion that the orchestrator can verify.

<p align="right"><a href="#readme-top">back to top</a></p>

<!-- LICENSE -->
## License

Distributed under the MIT License. See `LICENSE` for more information.

<p align="right"><a href="#readme-top">back to top</a></p>

<!-- CONTACT -->
## Contact

Andy — [@tuge](https://github.com/tuge)

Project Link: [https://github.com/tuge/tuge-skills](https://github.com/tuge/tuge-skills)

<p align="right"><a href="#readme-top">back to top</a></p>
