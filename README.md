# Introduction to Nextflow — Training Environment Setup

Welcome! This training follows the ["Hello Nextflow"](https://training.nextflow.io/latest/hello_nextflow/00_orientation/) course from the official Nextflow training materials. This README explains how to set up your training environment using **GitHub Codespaces** — no local installation required.

## Prerequisites

- A free [GitHub account](https://github.com/). If you sign in with an account attached to an organization, note that it may have additional restrictions — a personal account is recommended.
- A modern web browser (Chrome, Firefox, Edge, or Safari).
- No prior installation of Nextflow, Docker, or any other software is required — everything is pre-configured in the Codespace.

## 1. Launch the Codespace

Click the button below to open the Nextflow training environment in GitHub Codespaces:

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/nextflow-io/training?quickstart=1&ref=3.6)

**Tip:** Open this link in a new tab (right-click → "Open link in new tab", or ctrl/cmd-click) so you can keep these instructions visible while your environment loads.

### Configuration options

For this training, you generally don't need to change anything — just click the main button to launch with default settings. If you click **"Change options"**, you'll see:

- **Branch** — leave as the default unless instructed otherwise. `master` contains the latest materials, which may include unreleased changes.
- **Machine type** — the default (2 cores, 8 GB RAM, 32 GB storage) is sufficient for this course. Choosing a larger machine will consume your free quota faster.

### Startup time

The first launch can take a few minutes while your virtual machine is provisioned. This is normal and should not take more than five minutes.

## 2. Get familiar with the interface

Once loaded, the Codespace opens a browser-based **VS Code** interface with three main areas:

- **Sidebar (left):** file explorer, search, git, and other tools.
- **Main editor (center):** where code and files open for editing. It initially shows a preview of the repository `README.md`.
- **Terminal (bottom):** where you'll run all training commands.

## 3. Set your working directory

By default, the Codespace opens at the root of all training courses. For this course, move into the `hello-nextflow/` directory:

```bash
cd hello-nextflow/
```

Optionally, focus VS Code on this folder so only relevant files appear in the sidebar:

```bash
code .
```

**Tip:** If your Codespace goes to sleep and you lose your place, you can always get back with:

```bash
cd /workspaces/training/hello-nextflow
```

## 4. Explore the provided materials

Take a look at the directory contents:

```bash
tree . -L 2
```

You should see something like:

```
.
├── data
│   └── greetings.csv
├── hello-channels.nf
├── hello-config.nf
├── hello-containers.nf
├── hello-modules.nf
├── hello-workflow.nf
├── hello-world.nf
├── nextflow.config
├── solutions
│   ├── 1-hello-world
│   ├── 2-hello-channels
│   ├── 3-hello-workflow
│   ├── 4-hello-modules
│   ├── 5-hello-containers
│   └── 6-hello-config
├── test-params.json
└── test-params.yaml
```

- **`.nf` files** — workflow scripts, one per course section.
- **`nextflow.config`** — minimal environment configuration (ignore for now).
- **`data/greetings.csv`** — sample input data, introduced in Part 2 (Channels).
- **`test-params.*`** — configuration files used in Part 6 (Configuration).
- **`solutions/`** — completed reference scripts for each part of the course, useful for checking your work.

## 5. Version requirements

This training requires **Nextflow 25.10.2 or later**, with the **v2 syntax parser enabled**. The provided Codespace is pre-configured with the correct version — you don't need to do anything unless you're using a local/manual setup instead (see [Nextflow versions](https://training.nextflow.io/latest/info/nxf_versions/)).

## 6. Readiness checklist

Before starting the course, confirm:

- [ ] I have a GitHub account and can launch the Codespace.
- [ ] My Codespace is up and running.
- [ ] I've run `cd hello-nextflow/` and confirmed I'm in the right directory.
- [ ] I've explored the directory contents with `tree . -L 2`.

If all boxes are checked, you're ready to begin with [Part 1: Hello World](https://training.nextflow.io/latest/hello_nextflow/01_hello_world/).

## Notes on using GitHub Codespaces

- **Session timeout:** Your environment times out after 30 minutes of inactivity, but changes are saved for up to 2 weeks. Resume anytime from <https://github.com/codespaces/>.
- **Quotas:** Free accounts get up to 15 GB-month storage and 120 core-hours per month (~60 hours on the default 2-core machine). Avoid selecting a larger machine unless necessary, as it consumes quota faster.
- **Saving files locally:** Right-click any file in the explorer and select **Download**.

## Getting help

- [Community forum](https://community.seqera.io/c/training/39)
- [Nextflow Slack](https://www.nextflow.io/slack-invite.html)
- [GitHub issues](https://github.com/nextflow-io/training/issues)
- [Seqera AI assistant](https://seqera.io/ask-ai/chat-v2)
- [Nextflow Docs](https://nextflow.io/docs/latest/index.html)

## Full course materials

- [Hello Nextflow — Getting Started](https://training.nextflow.io/latest/hello_nextflow/00_orientation/)
- [Environment options (alternatives to Codespaces)](https://training.nextflow.io/latest/envsetup/)
