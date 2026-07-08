# Hello Nextflow — Part 2: Hello Channels

This README summarizes **Part 2: Hello Channels** of the [Hello Nextflow training course](https://training.nextflow.io/3.6/hello_nextflow/02_hello_channels/). It introduces Nextflow **channels** — the queues that shuttle data between workflow steps — and the **operators** used to transform their contents.

> **Prerequisite:** Complete [Part 1: Hello World](./README_part1_hello_world.md) first, or at least be comfortable with the basics of a `process` and `workflow` block.

## Learning objectives

By the end of this part, you will be able to:

- Explain what a Nextflow channel is and why it's needed for realistic workflows
- Create a channel explicitly with `channel.of()` and feed it into a process
- Inspect channel contents with `view()`
- Run a process over multiple input values and understand implicit parallelism
- Generate dynamic, unique output filenames to avoid overwriting results
- Use `flatten()` to unpack an array into individual channel elements
- Read inputs from a CSV file with `channel.fromPath()`, `splitCsv()`, and `map()`

---

## 0. Warmup: run `hello-channels.nf`

`hello-channels.nf` is the same as the finished script from Part 1, just publishing to a different subfolder. Confirm it works before making changes:

```bash
nextflow run hello-channels.nf --input 'Hello Channels!'
```

Check that `results/hello_channels/output.txt` contains `Hello Channels!`.

---

## 1. Provide variable inputs via a channel explicitly

In Part 1, the input was passed directly: `sayHello(params.input)`. This only works for a single value at a time. **Channels** solve that limitation.

### 1.1. Create an input channel

Use the [`channel.of()`](https://nextflow.io/docs/latest/reference/channel.html#of) factory to create a simple queue channel:

```groovy
greeting_ch = channel.of('Hello Channels!')
```

### 1.2. Feed the channel into the process

Replace the direct parameter with the channel:

```groovy
workflow {
    main:
    greeting_ch = channel.of('Hello Channels!')
    sayHello(greeting_ch)

    publish:
    first_output = sayHello.out
}
```

### 1.3. Run it

```bash
nextflow run hello-channels.nf
```

Functionally equivalent to before, but now built on the more flexible channel mechanism.

### 1.4. Inspect channel contents with `view()`

[`view()`](https://www.nextflow.io/docs/latest/reference/operator.html#view) is a debugging tool, like a `print()` statement, for peeking inside a channel:

```groovy
greeting_ch = channel.of('Hello Channels!')
    .view()
```

Run again — the channel's contents are printed to the console, one element per line.

**Takeaway:** you can use a basic channel factory to provide input to a process.

---

## 2. Run the workflow on multiple input values

### 2.1. Load multiple greetings

`channel.of()` happily accepts several values:

```groovy
greeting_ch = channel.of('Hello', 'Bonjour', 'Hola')
    .view()
```

```bash
nextflow run hello-channels.nf
```

The execution monitor shows `3 of 3` calls to `sayHello`. By default, Nextflow's condensed ANSI logging collapses all calls to a process onto a single summary line. To see every individual process call:

```bash
nextflow run hello-channels.nf -ansi-log false
```

This lists each submitted process call with its own work subdirectory.

> **Important:** with the condensed log, status (✔/✘) is shown; with `-ansi-log false`, only submission is reported, not completion status.

### 2.2. Ensure output filenames are unique

Because the output filename (`output.txt`) was hardcoded, all three parallel calls produce a file with the same name — so when published to the same `results/` folder, each overwrites the last.

**Fix:** build the filename dynamically from the input value:

```groovy
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

> **Syntax note:** the output filename expression **must** use double quotes (`"${greeting}-output.txt"`), not single quotes, or it will fail.

Run again — now `results/hello_channels/` contains `Hello-output.txt`, `Bonjour-output.txt`, and `Hola-output.txt`, all preserved.

> **In practice**, naming files from the raw input value is rarely practical for real pipelines. The standard approach is to pass structured metadata (e.g. from a sample sheet) alongside each input — covered later in the [Metadata side quest](https://training.nextflow.io/3.6/side_quests/metadata/).

**Takeaway:** you know how to feed multiple input elements through a channel.

---

## 3. Provide multiple inputs via an array

What if greetings start out bundled in an array rather than as separate arguments?

```groovy
greetings_array = ['Hello', 'Bonjour', 'Hola']
greeting_ch = channel.of(greetings_array)
    .view()
```

### 3.1. The naive attempt fails

Running this produces an error — Nextflow treats the *entire array* as a single value, so it tries to run one process call with `[Hello, Bonjour, Hola]` as a literal string, and the resulting filename is invalid.

### 3.2. Unpack the array with `flatten()`

[`flatten()`](https://nextflow.io/docs/latest/reference/operator.html#flatten) unpacks an array into individual channel elements:

```groovy
greeting_ch = channel.of(greetings_array)
    .view { greeting -> "Before flatten: $greeting" }
    .flatten()
    .view { greeting -> "After flatten: $greeting" }
```

Here, `{ greeting -> "..." }` is a **closure**: code executed once per channel item, with `greeting` as a temporary variable scoped to that closure. (You may see the implicit `$it` variable in older pipelines — explicit closure parameters are preferred and `$it` is being phased out.)

Run it — you'll see one `Before flatten:` line (the whole array) followed by three `After flatten:` lines (each individual greeting), and the workflow now succeeds.

> **Alternative:** [`channel.fromList()`](https://nextflow.io/docs/latest/reference/channel.html#fromlist) achieves the same result with an implicit flattening step built in.

**Takeaway:** you know how to use an operator like `flatten()` to transform channel contents, and how to use `view()` to compare before/after states.

---

## 4. Read input values from a CSV file

Realistic pipelines usually read structured input from files rather than hardcoded arrays. The provided `data/greetings.csv` contains:

```
Hello,English,123
Bonjour,French,456
Hola,Spanish,789
```

### 4.1. Point the pipeline at the CSV and switch channel factories

**a) Update the default parameter** to point at the CSV file:

```groovy
params {
    input: Path = 'data/greetings.csv'
}
```

**b) Switch from `channel.of()` to [`channel.fromPath()`](https://nextflow.io/docs/latest/reference/channel.html#frompath)**, which is designed to handle file paths:

```groovy
greeting_ch = channel.fromPath(params.input)
    .view { greeting -> "Before flatten: $greeting" }
```

Running this fails too — `channel.fromPath()` resolves the file's *path*, but doesn't read its *contents*. Nextflow tries to use the literal path string as the greeting.

### 4.2. Parse the file with `splitCsv()`

[`splitCsv()`](https://nextflow.io/docs/latest/reference/operator.html#splitcsv) parses CSV-formatted text into rows:

```groovy
greeting_ch = channel.fromPath(params.input)
    .view { csv -> "Before splitCsv: $csv" }
    .splitCsv()
    .view { csv -> "After splitCsv: $csv" }
```

This now parses the file correctly, but each row is emitted as a full array (e.g. `[Hello, English, 123]`), and the process still fails — we need just the greeting, not the whole row.

### 4.3. Extract a single column with `map()`

[`map()`](https://nextflow.io/docs/latest/reference/operator.html#map) transforms each channel element individually. Here, extract the first item of each row:

```groovy
greeting_ch = channel.fromPath(params.input)
    .view { csv -> "Before splitCsv: $csv" }
    .splitCsv()
    .view { csv -> "After splitCsv: $csv" }
    .map { item -> item[0] }
    .view { csv -> "After map: $csv" }
```

Run it:

```bash
nextflow run hello-channels.nf
```

Now each greeting is correctly extracted as an individual string, and `results/hello_channels/` contains one output file per greeting — with no code changes needed to add more rows to the CSV in future.

**Takeaway:** you know how to use `channel.fromPath()` with `splitCsv()` and `map()` to read inputs from a file. More broadly, you understand how channels manage inputs and how operators transform their contents — including Nextflow's implicit parallel execution over channel elements.

---

## Summary checklist

- [ ] I can explain what a channel is and why it's needed for multi-input workflows
- [ ] I created a channel with `channel.of()` and fed it into a process
- [ ] I used `view()` to inspect channel contents
- [ ] I ran a process over multiple values and understand why output filenames must be dynamic
- [ ] I used `flatten()` to unpack an array into individual channel elements
- [ ] I used `channel.fromPath()`, `splitCsv()`, and `map()` to read greetings from a CSV file

## Key operators covered

| Operator | Purpose |
|---|---|
| `view()` | Inspect/debug channel contents without modifying them |
| `flatten()` | Unpack an array into individual channel elements |
| `splitCsv()` | Parse CSV-formatted text into row arrays |
| `map()` | Transform each channel element (e.g. extract one field) |

## Next steps

Continue to [Part 3: Hello Workflow](https://training.nextflow.io/3.6/hello_nextflow/03_hello_workflow/) to learn how to add more steps and connect them into a proper multi-step workflow.

## Resources

- [Community forum](https://community.seqera.io/c/training/39)
- [Nextflow Slack](https://www.nextflow.io/slack-invite.html)
- [GitHub issues](https://github.com/nextflow-io/training/issues)
- [Seqera AI assistant](https://seqera.io/ask-ai/chat-v2)
- [Nextflow Docs](https://nextflow.io/docs/latest/index.html)
