# Hello Nextflow — Part 4: Hello Modules

This README summarizes **Part 4: Hello Modules** of the [Hello Nextflow training course](https://training.nextflow.io/3.6/hello_nextflow/04_hello_modules/). It covers how to organize workflow code into reusable **modules** for easier development and maintenance.

> **Prerequisite:** Complete [Part 1: Hello World](./README_part1_hello_world.md), [Part 2: Hello Channels](./README_part2_hello_channels.md), and [Part 3: Hello Workflow](./README_part3_hello_workflow.md) first, or be comfortable with the basics covered there.

## Learning objectives

By the end of this part, you will be able to:

- Explain what a Nextflow module is
- Extract a process from a workflow file into its own local module file
- Use an `include` declaration to pull a process back into a workflow
- Confirm that modularizing code doesn't break `-resume` caching

## What's a module?

In Nextflow, a **module** is a standalone code file, often encapsulating a single process definition. To use it in a workflow, you add a single-line `include` statement to your workflow file, then call the process as normal. This makes it possible to reuse process definitions across multiple workflows without duplicating code — if you later improve the module, every pipeline that includes it inherits the improvement.

So far, everything has lived in one script. In this part, we move each process (`sayHello`, `convertToUpper`, `collectGreetings`) out into its own file under a `modules/` directory.

---

## 0. Warmup: run `hello-modules.nf`

`hello-modules.nf` is equivalent to the Part 3 solution, just publishing to a new `hello_modules` subfolder. Confirm it works before making changes:

```bash
nextflow run hello-modules.nf
```

Check that `results/hello_modules/` contains the greeting, uppercase, collected, and report files as before.

---

## 1. Create a directory to store modules

By convention, modules live in a `modules/` directory:

```bash
mkdir modules
```

---

## 2. Create a module for `sayHello()`

Turning an existing process into a module is essentially copy-paste, plus one `include` line.

### 2.1. Create a file stub

```bash
touch modules/sayHello.nf
```

### 2.2. Move the process code into the module

Copy the entire `sayHello` process definition from `hello-modules.nf` into `modules/sayHello.nf`:

```groovy
/*
 * Use echo to print 'Hello World!' to a file
 */
process sayHello {

    input:
    val greeting

    output:
    path "${greeting}-output.txt"

    script:
    """
    echo '${greeting}' > '${greeting}-output.txt'
    """
}
```

Then **delete** the process definition from the workflow file — it now lives only in the module.

### 2.3. Add an `include` declaration

Syntax:

```groovy
include { <PROCESS_NAME> } from '<path_to_module>'
```

Insert it above the `params` block in `hello-modules.nf`:

```groovy
// Include modules
include { sayHello } from './modules/sayHello.nf'

/*
 * Pipeline parameters
 */
params {
    input: Path = 'data/greetings.csv'
    batch: String = 'batch'
}
```

### 2.4. Run the workflow

Since the code and inputs are functionally unchanged, run with `-resume`:

```bash
nextflow run hello-modules.nf -resume
```

All three processes should show `cached:` and complete almost instantly — Nextflow recognizes it's the same work, even though the code is now split across files.

**Takeaway:** you know how to extract a process into a local module without breaking the workflow's resumability.

---

## 3. Modularize the `convertToUpper()` process

Same pattern as before.

### 3.1. Create a file stub

```bash
touch modules/convertToUpper.nf
```

### 3.2. Move the process code

```groovy
/*
 * Use a text replacement tool to convert the greeting to uppercase
 */
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

Delete the corresponding process definition from `hello-modules.nf`.

### 3.3. Add the `include` declaration

```groovy
// Include modules
include { sayHello } from './modules/sayHello.nf'
include { convertToUpper } from './modules/convertToUpper.nf'

/*
 * Pipeline parameters
 */
params {
    input: Path = 'data/greetings.csv'
    batch: String = 'batch'
}
```

### 3.4. Run the workflow again

```bash
nextflow run hello-modules.nf -resume
```

All three processes should again show as cached, producing identical output.

---

## 4. Modularize the `collectGreetings()` process

Last one!

### 4.1. Create a file stub

```bash
touch modules/collectGreetings.nf
```

### 4.2. Move the process code

```groovy
/*
 * Collect uppercase greetings into a single output file
 */
process collectGreetings {

    input:
    path input_files
    val batch_name

    output:
    path "COLLECTED-${batch_name}-output.txt", emit: outfile
    path "${batch_name}-report.txt", emit: report

    script:
    count_greetings = input_files.size()
    """
    cat ${input_files} > 'COLLECTED-${batch_name}-output.txt'
    echo 'There were ${count_greetings} greetings in this batch.' > '${batch_name}-report.txt'
    """
}
```

Delete the corresponding process definition from `hello-modules.nf`.

### 4.3. Add the `include` declaration

```groovy
// Include modules
include { sayHello } from './modules/sayHello.nf'
include { convertToUpper } from './modules/convertToUpper.nf'
include { collectGreetings } from './modules/collectGreetings.nf'

/*
 * Pipeline parameters
 */
params {
    input: Path = 'data/greetings.csv'
    batch: String = 'batch'
}
```

### 4.4. Run the workflow

```bash
nextflow run hello-modules.nf -resume
```

Again, all three processes should show as cached, with identical output to before.

**Takeaway:** you know how to modularize multiple processes in a workflow. Functionally nothing has changed — but the code is now more modular, and any future pipeline that needs one of these processes can pull it in with a single `include` line instead of copy-pasting.

---

## Summary checklist

- [ ] I can explain what a module is and why it's useful
- [ ] I created a `modules/` directory
- [ ] I extracted `sayHello`, `convertToUpper`, and `collectGreetings` into their own module files
- [ ] I added `include` declarations for each module above the `params` block
- [ ] I confirmed `-resume` still works correctly after modularizing

## Key concepts covered

| Concept | Purpose |
|---|---|
| Module | A standalone file containing one or more process definitions |
| `modules/` directory | Conventional location for storing module files |
| `include { NAME } from './path'` | Pull a process definition into a workflow file |
| `-resume` after modularizing | Confirms that splitting code into files doesn't affect the execution cache |

## Next steps

Take a short break — then move on to [Part 5: Hello Containers](./README_part5_hello_containers.md) to learn how to use containers to manage software dependencies more conveniently and reproducibly.

## Resources

- [Community forum](https://community.seqera.io/c/training/39)
- [Nextflow Slack](https://www.nextflow.io/slack-invite.html)
- [GitHub issues](https://github.com/nextflow-io/training/issues)
- [Seqera AI assistant](https://seqera.io/ask-ai/chat-v2)
- [Nextflow Docs](https://nextflow.io/docs/latest/index.html)
- [Nextflow module documentation](https://nextflow.io/docs/latest/module.html)
