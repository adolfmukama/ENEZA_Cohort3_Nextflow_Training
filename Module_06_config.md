# Hello Nextflow — Part 6: Hello Config

This README summarizes **Part 6: Hello Config** of the [Hello Nextflow training course](https://training.nextflow.io/3.6/hello_nextflow/06_hello_config/) — the final part of the Hello Nextflow series. It covers how to configure a pipeline's behavior — inputs, outputs, software packaging, execution platform, resources, and profiles — **without changing any workflow code**.

> **Prerequisite:** Complete Parts 1–5 of Hello Nextflow first. If starting fresh from here, copy over the modules and config from the Part 5 solution:
> ```bash
> cp -r solutions/5-hello-containers/modules .
> cp solutions/5-hello-containers/nextflow.config .
> ```
> (`nextflow.config` should contain `docker.enabled = true`.)

## Learning objectives

By the end of this part, you will be able to:

- Manage workflow input parameters via `nextflow.config`, run-specific config files, and parameter files
- Control where and how workflow outputs are published, including dynamic output paths
- Choose between software packaging technologies (Docker, Conda) per process
- Select and configure an execution platform (local, HPC schedulers, cloud)
- Set and limit compute resource allocations, globally and per-process
- Use **profiles** to switch between preset configurations at runtime
- Use `nextflow config` to inspect the fully resolved configuration

Nextflow supports [multiple configuration mechanisms](https://nextflow.io/docs/latest/config.html) that combine according to a defined order of precedence. This part focuses on the most common one: the `nextflow.config` file, first introduced in Part 5.

---

## 0. Warmup: run `hello-config.nf`

Equivalent to the Part 5 solution, just with output destinations under `hello_config/`. Confirm it works first:

```bash
nextflow run hello-config.nf
```

Check `results/hello_config/cowpy-COLLECTED-batch-output.txt` for the expected ASCII art.

---

## 1. Manage workflow input parameters

There are several ways to override default parameter values without editing the workflow script itself.

### 1.1. Move default values to `nextflow.config`

**a) Add a `params` block to `nextflow.config`** (note: value *assignments*, not typed declarations):

```groovy
docker.enabled = true

params {
    input = 'data/greetings.csv'
    batch = 'batch'
    character = 'turkey'
}
```

**b) Remove the default values from the `params` block in the workflow file**, leaving only typed declarations:

```groovy
params {
    input: Path
    batch: String
    character: String
}
```

Run `nextflow run hello-config.nf` — output is unchanged, but defaults now live in the config file rather than the script.

### 1.2. Use a run-specific configuration file

For quick experiments, create a subdirectory with its own blank config so you don't touch the main one:

```bash
mkdir -p tux-run
cd tux-run
touch nextflow.config
```

Populate `tux-run/nextflow.config` (note the relative path to input data):

```groovy
params {
    input = '../data/greetings.csv'
    batch = 'experiment'
    character = 'tux'
}
```

Run from within the new directory:

```bash
nextflow run ../hello-config.nf
```

Nextflow merges this local config with the pipeline's root `nextflow.config`, so `tux` overrides the default `turkey` character. Outputs land under `tux-run/results/`.

> **Remember to `cd ..`** back to the main directory before continuing.

### 1.3. Use a parameter file

For distributing a specific set of parameter values (especially useful for pipelines with many parameters), use a YAML or JSON [parameter file](https://nextflow.io/docs/latest/config.html#params-file). Example `test-params.yaml`:

```yaml
input: "data/greetings.csv"
batch: "yaml"
character: "stegosaurus"
```

Note: colons (`:`), not equals signs — this is YAML, not Groovy. Run with:

```bash
nextflow run hello-config.nf -params-file test-params.yaml
```

This is convenient for sharing reproducible parameter sets with collaborators, e.g. alongside a publication, without long command lines.

**Takeaway:** you know how to manage workflow inputs via the main config, a run-specific config, or a parameter file.

---

## 2. Manage workflow outputs

Two independent concerns: the **top-level output directory**, and **how files are organized within it**.

### 2.1. Customize the output directory with `-output-dir`

The `-output-dir` (or `-o`) CLI option overrides the default `results/` directory for all outputs — the recommended way to set the root output path:

```bash
nextflow run hello-config.nf -output-dir custom-outdir-cli/
```

If your `output {}` block still hardcodes a `hello_config` subdirectory prefix in each `path`, remove it now that output location is handled via configuration — set `path` to `''` or omit it for outputs that don't need a subdirectory.

### 2.2. Dynamic output paths

**a) Set `outputDir` in `nextflow.config`**, optionally built dynamically from a parameter:

```groovy
outputDir = "custom-outdir-config/${params.batch}"
```

Now `--batch my_run` changes both the batch name and the output subdirectory in one go.

> **Precedence:** `-output-dir` on the CLI always overrides `outputDir` in the config — if set, the config value is ignored entirely.

**b) Build dynamic subdirectories per output**, referencing `<process>.name`, so files are grouped by the process that produced them:

```groovy
output {
    first_output {
        path { "${params.batch}/intermediates/${sayHello.name}" }
        mode 'copy'
    }
    uppercased {
        path { "${params.batch}/intermediates/${convertToUpper.name}" }
        mode 'copy'
    }
    collected {
        path { "${params.batch}/intermediates/${collectGreetings.name}" }
        mode 'copy'
    }
    batch_report {
        path { "${params.batch}/${collectGreetings.name}" }
        mode 'copy'
    }
    cowpy_art {
        path { "${params.batch}/${cowpy.name}" }
        mode 'copy'
    }
}
```

> **Tip:** putting `params.batch` in the `path` declaration (rather than in `outputDir`) means it survives even if `-output-dir` overrides the root path on the CLI.

Run with both a custom root and batch name:

```bash
nextflow run hello-config.nf -output-dir custom-outdir-config-2 --batch rep2
```

Result: outputs land under `custom-outdir-config-2/rep2/`, organized into per-process subfolders.

### 2.3. Set the publish mode at the workflow level

Instead of repeating `mode 'copy'` in every output block entry, set it once in the config:

```groovy
workflow.output.mode = 'copy'
```

Then remove the per-output `mode 'copy'` lines from the workflow file's `output {}` block — Nextflow applies the config-level setting to all of them. You'd only keep per-output `mode` if you want to mix copying and symlinking within the same run.

**Takeaway:** you know how to control the naming and structure of published output directories, and set the publish mode centrally.

---

## 3. Select a software packaging technology

So far, Docker has handled the `cowpy` process's software dependency. Some environments (e.g. HPC clusters where Docker is disallowed) require an alternative like [Conda](https://nextflow.io/docs/latest/conda.html).

### 3.1. Disable Docker and enable Conda

```groovy
docker.enabled = false
conda.enabled = true
```

### 3.2. Specify a Conda package in the process definition

Add a `conda` directive to `modules/cowpy.nf`, *alongside* (not replacing) the existing `container` directive:

```groovy
process cowpy {

    container 'community.wave.seqera.io/library/cowpy:1.1.5--3db457ae1977a273'
    conda 'conda-forge::cowpy==1.1.5'

    input:
    ...
```

> **Tip:** [Seqera Containers](https://seqera.io/containers/) is a convenient way to look up Conda package URIs, even when you're not building a container.

### 3.3. Run and verify

```bash
nextflow run hello-config.nf --batch conda
```

Nextflow builds and caches a Conda environment (`Creating env using conda: ...`) the first time — this may take a minute for larger packages, but subsequent runs reuse the cached environment. Output is functionally identical to the Docker run.

> **Mixing technologies:** because `container` and `conda` directives are set per process, you can enable both Docker and Conda globally and let different processes use different technologies. If both are available for a given process, Nextflow prioritizes containers.

**Takeaway:** you know how to configure and switch between software packaging technologies per process.

---

## 4. Select an execution platform

By default, Nextflow uses the **local executor** — running tasks on the same machine, queuing tasks when CPU/memory demand exceeds what's available. For larger workloads, Nextflow supports [many executors](https://nextflow.io/docs/latest/executor.html): HPC schedulers (Slurm, LSF, SGE, PBS, and others) and cloud backends (AWS Batch, Google Cloud Batch, Azure Batch, Kubernetes, etc.).

### 4.1. Targeting a different backend

The executor is a process directive. Default (implied):

```groovy
process {
    executor = 'local'
}
```

To target Slurm, for example:

```groovy
process {
    executor = 'slurm'
}
```

> **Note:** this cannot actually be tested in a Codespaces environment without HPC access.

### 4.2. Backend-specific execution syntax

Each scheduler (Slurm, PBS, SGE, ...) has its own syntax for resource requests. Nextflow abstracts this away: you specify standardized directives like [`cpus`](https://nextflow.io/docs/latest/reference/process.html#cpus), [`memory`](https://nextflow.io/docs/latest/reference/process.html#memory), and [`queue`](https://nextflow.io/docs/latest/reference/process.html#queue) once, and Nextflow translates them into the correct backend-specific submission script at runtime.

**Takeaway:** you know how to change the executor to target different computing infrastructure.

---

## 5. Control compute resource allocations

Default resource allocation (implied):

```groovy
process {
    cpus = 1
    memory = 2.GB
}
```

### 5.1. Generate a resource utilization report

Profile actual usage before tuning allocations:

```bash
nextflow run hello-config.nf -with-report report-config-1.html
```

Open the generated HTML report (or preview it in VS Code) to see CPU/memory utilization as a percentage of what was allocated, and identify processes that are over- or under-provisioned.

### 5.2. Set resource allocations for all processes

Based on profiling, reduce the default memory:

```groovy
process {
    memory = 1.GB
}
```

### 5.3. Set resource allocations for a specific process

Override for one process using `withName`:

```groovy
process {
    memory = 1.GB
    withName: 'cowpy' {
        memory = 2.GB
        cpus = 2
    }
}
```

All processes get 1GB/1 CPU except `cowpy`, which gets 2GB/2 CPUs.

> **Tip:** if your machine has few CPUs and you allocate a high number per process, task calls may queue behind each other — Nextflow won't exceed available resources.

### 5.4. Run with the updated configuration

```bash
nextflow run hello-config.nf -with-report report-config-2.html
```

Compare the two reports to evaluate the effect of your changes. On a tiny workload like this you likely won't see a real difference, but this is the standard workflow for right-sizing resources on real pipelines using data instead of guesswork.

> Nextflow also has built-in [dynamic retry logic](https://nextflow.io/docs/latest/process.html#dynamic-task-resources) to automatically retry failed jobs with increased resources.

### 5.5. Add resource limits

Cap allocations to stay within cluster-imposed constraints using `resourceLimits`:

```groovy
process {
    resourceLimits = [
        memory: 750.GB,
        cpus: 200,
        time: 30.d
    ]
}
```

Nextflow translates this into the appropriate submission parameters for your executor, capping any request that exceeds these limits. (Not testable without HPC access in this training environment.)

> **Tip:** the [nf-core institutional configs collection](https://nf-co.re/configs/) has ready-made configuration files shared by research institutions worldwide — useful both directly and as reference examples.

**Takeaway:** you know how to profile resource usage and set global, per-process, and limit-based resource allocations.

---

## 6. Use profiles to switch between preset configurations

[Profiles](https://nextflow.io/docs/latest/config.html#config-profiles) let you predefine alternative configurations and select them at runtime with a CLI flag, rather than editing the config file.

### 6.1. Profiles for local development vs. HPC

Add a `profiles` block to `nextflow.config`:

```groovy
profiles {
    my_laptop {
        process.executor = 'local'
        docker.enabled = true
    }
    univ_hpc {
        process.executor = 'slurm'
        conda.enabled = true
        process.resourceLimits = [
            memory: 750.GB,
            cpus: 200,
            time: 30.d
        ]
    }
}
```

Select a profile at runtime:

```bash
nextflow run hello-config.nf -profile my_laptop
```

> `univ_hpc` won't actually run in this training environment since there's no Slurm scheduler available — it's illustrative.

### 6.2. Create a profile of test parameters

Profiles can also set default parameter values — a lightweight alternative to a parameter file, handy for letting others try the workflow without gathering their own inputs:

```groovy
profiles {
    my_laptop { ... }
    univ_hpc { ... }
    test {
        params.input = 'data/greetings.csv'
        params.batch = 'test'
        params.character = 'dragonandcow'
    }
}
```

Profiles are **not mutually exclusive** — combine them with a comma-separated list:

```bash
nextflow run hello-config.nf -profile my_laptop,test
```

If profiles conflict on the same setting within one config file, the one read **last** in the file wins; across different config sources, the standard [order of precedence](https://nextflow.io/docs/latest/config.html) applies.

> **Tip:** for larger input files, point to a URL — Nextflow will download it automatically as long as there's a live connection. See the [Working with Files side quest](https://training.nextflow.io/3.6/side_quests/working_with_files/).

### 6.3. Use `nextflow config` to see the resolved configuration

With config coming from multiple possible sources, use the built-in `config` command to see exactly what Nextflow will use, without launching anything:

```bash
nextflow config
```

This prints the fully merged `params`, `docker`, `conda`, and other blocks as Nextflow would resolve them. You can also apply `-profile` flags to `nextflow config` to preview the resolved configuration for a specific combination of profiles before running.

**Takeaway:** you know how to define and combine configuration profiles, and how to inspect the fully resolved configuration with `nextflow config`.

---

## Summary checklist

- [ ] I moved default parameter values into `nextflow.config`
- [ ] I used a run-specific config file and a YAML parameter file to override parameters
- [ ] I customized the output directory with `-output-dir` and dynamic `outputDir` / `path` expressions
- [ ] I set the publish mode centrally with `workflow.output.mode`
- [ ] I switched the `cowpy` process between Docker and Conda
- [ ] I understand how to target a different executor (Slurm, cloud, etc.)
- [ ] I generated a resource utilization report and set global/per-process resource allocations and limits
- [ ] I created and combined configuration profiles, and inspected them with `nextflow config`

## Key configuration mechanisms covered

| Mechanism | Purpose |
|---|---|
| `params {}` in `nextflow.config` | Central default parameter values |
| Run-specific `nextflow.config` | Temporary experiments without touching the main config |
| `-params-file` | Distributable YAML/JSON parameter sets |
| `-output-dir` / `outputDir` | Control the root output directory |
| `path { }` (dynamic) in `output {}` | Organize outputs into subdirectories per batch/process |
| `workflow.output.mode` | Set publish mode (`copy`, symlink, etc.) once for all outputs |
| `docker.enabled` / `conda.enabled` + `container`/`conda` directives | Choose software packaging per process |
| `process.executor` | Target local, HPC, or cloud execution backends |
| `cpus`, `memory`, `withName` | Set resource allocations globally or per process |
| `resourceLimits` | Cap resource requests to cluster-imposed limits |
| `profiles {}` | Preset, switchable configuration bundles |
| `nextflow config` | Preview the fully resolved configuration |

## Next steps

This concludes the **Hello Nextflow** course. Consider taking the [course quiz](https://training.nextflow.io/3.6/hello_nextflow/06_hello_config/#quiz), then move on to [Hello nf-core](https://training.nextflow.io/3.6/hello_nf-core/) to learn how to work with the nf-core pipeline ecosystem, or explore the [Nextflow for Science](https://training.nextflow.io/3.6/nf4_science/) tracks (Genomics, RNAseq, Imaging).

## Resources

- [Community forum](https://community.seqera.io/c/training/39)
- [Nextflow Slack](https://www.nextflow.io/slack-invite.html)
- [GitHub issues](https://github.com/nextflow-io/training/issues)
- [Seqera AI assistant](https://seqera.io/ask-ai/chat-v2)
- [Nextflow Docs](https://nextflow.io/docs/latest/index.html)
- [Nextflow configuration documentation](https://nextflow.io/docs/latest/config.html)
- [nf-core institutional configs](https://nf-co.re/configs/)
