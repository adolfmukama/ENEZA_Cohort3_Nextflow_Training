# Hello Nextflow — Part 3: Hello Workflow

This README summarizes **Part 3: Hello Workflow** of the [Hello Nextflow training course](https://training.nextflow.io/3.6/hello_nextflow/03_hello_workflow/). It covers how to connect multiple processes together into a real multi-step workflow.

> **Prerequisite:** Complete [Part 1: Hello World](./README_part1_hello_world.md) and [Part 2: Hello Channels](./README_part2_hello_channels.md) first, or be comfortable with the basics covered there.

## Learning objectives

By the end of this part, you will be able to:

- Make data flow from one process to the next
- Collect outputs from multiple process calls into a single process call using `collect()`
- Pass additional parameters to a process
- Handle multiple named outputs coming out of a process

We continue building on the "Hello World" example, adding: a second step that uppercases each greeting, a third step that collects all greetings into one file, a parameter to name that output file, and a simple statistic reported by the collection step.

---

## 0. Warmup: run `hello-workflow.nf`

This starting script is equivalent to the finished result of Part 2, minus the `view()` calls and publishing to a new subfolder.

```bash
nextflow run hello-workflow.nf
```

Confirm `results/hello_workflow/` contains `Hello-output.txt`, `Bonjour-output.txt`, and `Hola-output.txt`.

---

## 1. Add a second step to the workflow

Goal: convert each greeting to uppercase using the classic UNIX tool `tr`.

### 1.1. Prototype the command in the terminal

```bash
echo 'Hello World' | tr '[a-z]' '[A-Z]' > UPPER-output.txt
```

### 1.2. Write the uppercasing process

```groovy
process convertToUpper {
    input:
    path input_file

    output:
    path "UPPER-${input_file}"

    script:
    """
    cat '${input_file}' | tr '[a-z]' '[A-Z]' > 'UPPER-${input_file}'
    """
}
```

### 1.3–1.4. Call it in the workflow, chained to the first process

Nextflow automatically packages a process's output into a channel accessible as `<process>.out`. Wiring two processes together is as simple as passing that channel to the next call:

```groovy
workflow {
    main:
    greeting_ch = channel.fromPath(params.input)
                        .splitCsv()
                        .map { line -> line[0] }
    sayHello(greeting_ch)
    convertToUpper(sayHello.out)

    publish:
    first_output = sayHello.out
}
```

### 1.5. Publish the new outputs

Add a second entry to both the `publish:` and `output {}` blocks:

```groovy
workflow {
    ...
    publish:
    first_output = sayHello.out
    uppercased = convertToUpper.out
}

output {
    first_output {
        path 'hello_workflow'
        mode 'copy'
    }
    uppercased {
        path 'hello_workflow'
        mode 'copy'
    }
}
```

### 1.6. Run with `-resume`

```bash
nextflow run hello-workflow.nf -resume
```

Since `sayHello` already ran successfully, its results are pulled from cache (`cached: 3 ✔`) while `convertToUpper` executes fresh. Inspecting a `convertToUpper` task directory shows the upstream output file staged in as a **symbolic link** — Nextflow's default way of passing files between process work directories on a single machine.

**Takeaway:** you know how to chain processes by feeding one process's output channel into the next process call.

---

## 2. Add a third step to collect all the greetings

Goal: concatenate all the uppercased greetings into a single file.

### 2.1. Prototype the command

```bash
echo 'Hello' | tr '[a-z]' '[A-Z]' > UPPER-Hello-output.txt
echo 'Bonjour' | tr '[a-z]' '[A-Z]' > UPPER-Bonjour-output.txt
echo 'Hola' | tr '[a-z]' '[A-Z]' > UPPER-Hola-output.txt
cat UPPER-Hello-output.txt UPPER-Bonjour-output.txt UPPER-Hola-output.txt > COLLECTED-output.txt
```

### 2.2. Write the collector process

The key challenge: the process must handle an arbitrary number of input files.

```groovy
process collectGreetings {
    input:
    path input_files

    output:
    path "COLLECTED-output.txt"

    script:
    """
    cat ${input_files} > 'COLLECTED-output.txt'
    """
}
```

Note that `path input_files` is declared singular even though it will receive multiple files — and `cat ${input_files}` expands to list every file automatically.

> **Tip:** some CLI tools require a repeated flag per input file (e.g. `-input file1 -input file2`). That needs extra command composition, covered in the [Nextflow for Genomics course](https://training.nextflow.io/3.6/nf4_science/genomics/).

### 2.3. Wire it up — the naive way (and why it fails)

```groovy
collectGreetings(convertToUpper.out)
```

Add to `publish:` and `output {}` as before. Running with `-resume` reveals a problem: `collectGreetings` runs **three times** — once per greeting — instead of once for all of them. The final file only contains the last greeting processed.

This happens because, without intervention, Nextflow calls a process once per channel element, exactly like it did for `sayHello` and `convertToUpper`.

### 2.4. Fix it with the `collect()` operator

[`collect()`](https://nextflow.io/docs/latest/reference/operator.html#collect) gathers all elements of a channel into a single list, packaged as one channel element:

```groovy
collectGreetings(convertToUpper.out.collect())
```

Temporarily add `view()` calls to see the effect:

```groovy
convertToUpper.out.view { contents -> "Before collect: $contents" }
convertToUpper.out.collect().view { contents -> "After collect: $contents" }
```

Re-running shows three `Before collect:` lines (individual file paths) followed by a single `After collect:` line (all three paths bundled into one list) — and `collectGreetings` now runs exactly once, correctly producing all three greetings in the output file.

> **Note:** without `-resume`, greeting order in the collected file is not guaranteed to be consistent between runs.

Remove the `view()` calls once you've confirmed it works, to keep console output clean.

**Takeaway:** you know how to collect outputs from a batch of process calls and feed them into a single downstream process using `collect()`.

---

## 3. Pass additional parameters to a process

Goal: let the user name the output batch (e.g. via `--batch trio`) so successive runs don't overwrite each other's results.

### 3.1. Add a second input to the collector process

```groovy
process collectGreetings {
    input:
    path input_files
    val batch_name

    output:
    path "COLLECTED-${batch_name}-output.txt"

    script:
    """
    cat ${input_files} > 'COLLECTED-${batch_name}-output.txt'
    """
}
```

Processes can declare as many inputs as needed — here all are treated as required.

### 3.2. Add a `--batch` CLI parameter

```groovy
params {
    input: Path = 'data/greetings.csv'
    batch: String = 'batch'
}
```

Pass it through in the process call:

```groovy
collectGreetings(convertToUpper.out.collect(), params.batch)
```

> **Warning:** inputs must be supplied to a process call in the **exact same order** they're declared in the `input:` block.

### 3.3. Run it

```bash
nextflow run hello-workflow.nf -resume --batch trio
```

Output now lands in `results/hello_workflow/COLLECTED-trio-output.txt`, so future batches (with a different `--batch` value) won't clobber this result.

**Takeaway:** you know how to pass more than one input to a process.

---

## 4. Add an output to the collector step

Goal: have the collector also report how many greetings it processed.

### 4.1. Count greetings and write a report

Nextflow lets you run arbitrary Groovy code in the `script:` block before the command string, e.g. using the built-in `.size()` function:

```groovy
output:
path "COLLECTED-${batch_name}-output.txt", emit: outfile
path "${batch_name}-report.txt", emit: report

script:
count_greetings = input_files.size()
"""
cat ${input_files} > 'COLLECTED-${batch_name}-output.txt'
echo 'There were ${count_greetings} greetings in this batch.' > '${batch_name}-report.txt'
"""
```

The `emit:` tag names each output channel so you can refer to it explicitly (e.g. `collectGreetings.out.outfile`, `collectGreetings.out.report`) instead of relying on positional indices like `collectGreetings.out[0]`. Named outputs are recommended — indices are easy to mix up once a process has several outputs.

### 4.2. Update workflow-level publishing

```groovy
workflow {
    ...
    publish:
    first_output = sayHello.out
    uppercased = convertToUpper.out
    collected = collectGreetings.out.outfile
    batch_report = collectGreetings.out.report
}

output {
    first_output   { path 'hello_workflow'; mode 'copy' }
    uppercased     { path 'hello_workflow'; mode 'copy' }
    collected      { path 'hello_workflow'; mode 'copy' }
    batch_report   { path 'hello_workflow'; mode 'copy' }
}
```

### 4.3. Run it

```bash
nextflow run hello-workflow.nf -resume --batch trio
```

You'll now find `results/hello_workflow/trio-report.txt` containing something like:

```
There were 3 greetings in this batch.
```

**Takeaway:** you know how to make a process emit multiple named outputs and handle them appropriately at the workflow level.

---

## Summary checklist

- [ ] I chained two processes by passing `<process>.out` as input to the next
- [ ] I understand why output filenames from a batch need dynamic naming
- [ ] I used `collect()` to gather all elements of a channel into one input
- [ ] I passed multiple ordered inputs to a process, including a CLI parameter
- [ ] I used `emit:` to name multiple outputs and reference them individually

## Key concepts covered

| Concept | Purpose |
|---|---|
| `<process>.out` | Access a process's output as a channel |
| `collect()` | Gather all channel elements into a single list element |
| Multiple `input:` declarations | Pass more than one value/file to a process (order matters) |
| `emit:` | Name a specific output channel for clear downstream reference |

## Next steps

Take a break — then continue to [Part 4: Hello Modules](https://training.nextflow.io/3.6/hello_nextflow/04_hello_modules/) to learn how to modularize your code for better maintainability and reuse.

## Resources

- [Community forum](https://community.seqera.io/c/training/39)
- [Nextflow Slack](https://www.nextflow.io/slack-invite.html)
- [GitHub issues](https://github.com/nextflow-io/training/issues)
- [Seqera AI assistant](https://seqera.io/ask-ai/chat-v2)
- [Nextflow Docs](https://nextflow.io/docs/latest/index.html)
