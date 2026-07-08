# Hello Nextflow — Part 1: Hello World

This README summarizes **Part 1: Hello World** of the [Hello Nextflow training course](https://training.nextflow.io/3.6/hello_nextflow/01_hello_world/). It introduces the basic building blocks of a Nextflow workflow using a minimal "Hello World" example.

> **Prerequisite:** Complete the [environment setup](./README.md) first and make sure you're working inside the `hello-nextflow/` directory.

## Learning objectives

By the end of this part, you will be able to:

- Explain the structure of a basic Nextflow script (`process` + `workflow`)
- Run a Nextflow workflow and locate its outputs and logs in the `work/` directory
- Publish workflow outputs to a convenient location
- Pass variable input to a workflow via command-line parameters, including default values
- Use `-resume` to re-run a workflow without repeating completed steps
- Inspect run history with `nextflow log` and clean up old runs with `nextflow clean`

---

## 0. Warmup: run Hello World directly in the terminal

Before touching Nextflow, reproduce the same result with plain shell commands:

```bash
echo 'Hello World!'
echo 'Hello World!' > output.txt
```

The second command writes the text to `output.txt` instead of printing it — this is what the Nextflow version will replicate.

---

## 1. Examine and run `hello-world.nf`

Open `hello-world.nf` in the editor. It contains two core building blocks:

**A `process`** — defines *what* operation to run:

```groovy
process sayHello {
    output:
    path 'output.txt'

    script:
    """
    echo 'Hello World!' > output.txt
    """
}
```

- `output:` *declares* the expected output so Nextflow can verify and pass it downstream — it does not create the file itself.
- `script:` contains the actual command to execute.

**A `workflow`** — defines *how* processes are connected:

```groovy
workflow {
    main:
    sayHello()
}
```

### Run it

```bash
nextflow run hello-world.nf
```

The console output ends with a line like:

```
[65/7be2fa] sayHello | 1 of 1 ✔
```

The bracketed hash (`65/7be2fa`) points to a subdirectory under `work/` where Nextflow stages inputs, executes the task, and stores logs and outputs.

### Explore the `work/` directory

```bash
tree -a work
```

Each task subdirectory contains:

| File | Contents |
|---|---|
| `.command.sh` | The actual command that was run |
| `.command.run` | Full wrapper script Nextflow used to execute the task |
| `.command.out` / `.command.err` | stdout / stderr |
| `.command.log` | Combined log output |
| `.command.begin` | Execution start metadata |
| `.exitcode` | Exit status of the command |
| `output.txt` | The task's actual output |

**Key takeaway:** each run creates a new, unique task subdirectory under `work/` — previous runs are never overwritten.

---

## 2. Publish outputs

Outputs buried under `work/` are inconvenient to retrieve. Nextflow's [workflow output definitions](https://nextflow.io/docs/latest/workflow.html#workflow-outputs) solve this with two additions:

**a) A `publish:` block** inside the workflow, naming the output:

```groovy
workflow {
    main:
    sayHello()

    publish:
    first_output = sayHello.out
}
```

**b) An `output {}` block**, outside and below the `workflow` block, defining where to write it:

```groovy
output {
    first_output {
        path '.'
    }
}
```

Run again:

```bash
nextflow run hello-world.nf
```

Nextflow now creates a `results/` directory and prints an `Outputs:` summary showing where each named output was published.

### Custom subdirectories

Change the `path` to organize outputs into subfolders:

```groovy
output {
    first_output {
        path 'hello_world'
    }
}
```

### Copy instead of symlink

By default, published files are **symbolic links** back to `work/` — convenient for large files, but they break if `work/` is deleted. Add `mode 'copy'` to store real copies:

```groovy
output {
    first_output {
        path 'hello_world'
        mode 'copy'
    }
}
```

> **Note:** You may see older pipelines use a process-level `publishDir 'results/hello_world', mode: 'copy'` directive instead. This still works but is deprecated in favor of workflow-level `output` blocks.

---

## 3. Use a variable input from the command line

Hardcoding the greeting isn't very flexible. Three changes make it configurable:

**a) Accept an input in the process:**

```groovy
process sayHello {
    input:
    val greeting

    output:
    path 'output.txt'

    script:
    """
    echo '${greeting}' > output.txt
    """
}
```

**b) Pass a CLI parameter to the process call in the workflow:**

```groovy
workflow {
    main:
    sayHello(params.input)
    ...
}
```

Nextflow's built-in [`params`](https://nextflow.io/docs/latest/config.html#params) system automatically maps `params.input` to a `--input` command-line flag.

**c) Run with a custom greeting:**

```bash
nextflow run hello-world.nf --input 'Bonjour le monde!'
```

### Set a default value

Declare parameters (with types, Nextflow 25.10.2+) before the `workflow` block:

```groovy
params {
    input: String = 'Hola mundo!'
}
```

Supported types: `String`, `Integer`, `Float`, `Boolean`, `Path`.

Now the workflow runs without requiring `--input`, but a CLI value will still override the default:

```bash
nextflow run hello-world.nf                        # uses default 'Hola mundo!'
nextflow run hello-world.nf --input 'Konnichiwa!'  # overrides default
```

> **Troubleshooting:** If you get a script compilation error mentioning `java.lang.String`, your Nextflow is using the older v1 language parser. Enable v2 with:
> ```bash
> export NXF_SYNTAX_PARSER=v2
> ```
> (v2 is the default from Nextflow 26.04 onward.)

> **Syntax reminder:** pipeline parameters use a double hyphen (`--input`); Nextflow-level options use a single hyphen (`-resume`).

---

## 4. Manage workflow executions

### Resume a previous run

Skip already-completed, unchanged steps with `-resume`:

```bash
nextflow run hello-world.nf --input 'Konnichiwa!' -resume
```

Look for `cached:` in the console output — it confirms Nextflow reused a previous result instead of re-running it. Note: `-resume` does **not** overwrite previously published files.

### Inspect run history

Every run is logged to `.nextflow/history`. View it more conveniently with:

```bash
nextflow log
```

This lists timestamp, duration, run name, status, revision ID, session ID, and the exact command for each run. The session ID stays constant across `-resume` runs but changes for fresh runs.

### Clean up old work directories

Use [`nextflow clean`](https://nextflow.io/docs/latest/reference/cli.html#clean) to remove old task directories:

```bash
# Dry run first — see what would be deleted
nextflow clean -before <run_name> -n

# Then actually delete
nextflow clean -before <run_name> -f
```

Replace `<run_name>` with a run name from `nextflow log` (e.g. `golden_cantor`).

> **Warning:** Deleting `work/` subdirectories removes them from the cache (breaking `-resume` for those steps) and deletes any outputs stored only there. Prefer `mode 'copy'` for anything you need to keep.

---

## Summary checklist

- [ ] I can explain the `process` and `workflow` blocks in `hello-world.nf`
- [ ] I ran the workflow and located its output in `work/`
- [ ] I published outputs to `results/` using `publish:` and `output {}`
- [ ] I customized the output path and set publish mode to `copy`
- [ ] I added a variable input via `params` and set a default value
- [ ] I used `-resume`, `nextflow log`, and `nextflow clean`

## Next steps

Continue to [Part 2: Hello Channels](https://training.nextflow.io/3.6/hello_nextflow/02_hello_channels/) to learn how channels feed inputs into workflows and enable Nextflow's dataflow parallelism.

## Resources

- [Community forum](https://community.seqera.io/c/training/39)
- [Nextflow Slack](https://www.nextflow.io/slack-invite.html)
- [GitHub issues](https://github.com/nextflow-io/training/issues)
- [Nextflow Docs](https://nextflow.io/docs/latest/index.html)
