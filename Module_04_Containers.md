# Hello Nextflow — Part 5: Hello Containers

This README summarizes **Part 5: Hello Containers** of the [Hello Nextflow training course](https://training.nextflow.io/3.6/hello_nextflow/05_hello_containers/). It introduces **containers** — lightweight, portable execution environments that solve the "it works on my machine" problem — and shows how Nextflow uses them to run process steps.

> **Prerequisite:** Complete Parts 1–4 of Hello Nextflow first, or have a working pipeline equivalent to the Part 4 solution. If jumping in fresh, copy over the modules directory:
> ```bash
> cp -r solutions/4-hello-modules/modules .
> ```

## Learning objectives

By the end of this part, you will be able to:

- Explain what a container is and how it differs from a container image
- Pull and run a container image manually with Docker, both one-off and interactively
- Mount host data into a container so tools inside it can access your files
- Add a `container` directive to a Nextflow process
- Enable Docker execution via `nextflow.config`
- Inspect how Nextflow launches a containerized task under the hood

We'll be using **Docker** for this training, though Nextflow also supports [several other container technologies](https://nextflow.io/docs/latest/container.html) (Singularity/Apptainer, Podman, etc.).

---

## 0. Warmup: run `hello-containers.nf`

`hello-containers.nf` is equivalent to the Part 4 solution, just publishing to a new subfolder:

```bash
nextflow run hello-containers.nf
```

Confirm `results/hello_containers/` contains the expected greeting, uppercase, collected, and report files.

---

## 1. Use a container "manually"

Before wiring containers into Nextflow, it helps to understand what's actually happening by using Docker directly.

### 1.1. Pull a container image

The general syntax:

```bash
docker pull '<container>'
```

For this exercise, we'll use a container built from the [`cowpy`](https://github.com/jeffbuttars/cowpy) Conda package (a Python implementation of `cowsay`, which renders text as ASCII art), published via [Seqera Containers](https://seqera.io/containers/):

```bash
docker pull 'community.wave.seqera.io/library/cowpy:1.1.5--3db457ae1977a273'
```

This downloads a local copy of the image (may take a minute on first pull).

### 1.2. Run `cowpy` as a one-off command

```bash
docker run --rm '<container>' [tool command]
```

`--rm` tells Docker to remove the container instance once the command finishes. Try it:

```bash
docker run --rm 'community.wave.seqera.io/library/cowpy:1.1.5--3db457ae1977a273' cowpy
```

Docker spins up the container, runs `cowpy`, prints the ASCII art to the console, then tears the instance down.

### 1.3. Run `cowpy` interactively

#### 1.3.1. Spin up the container

Add `-it` for an interactive shell, optionally specifying which shell to launch:

```bash
docker run --rm -it 'community.wave.seqera.io/library/cowpy:1.1.5--3db457ae1977a273' /bin/bash
```

Your prompt changes to something like `(base) root@<container-id>:/tmp#`, confirming you're now inside the container. Running `ls /` shows a filesystem completely separate from your host machine.

#### 1.3.2. Run tool commands

Try different characters using the `-c` flag:

```bash
echo "Hello Containers" | cowpy -c tux
```

Available characters include `beavis`, `cheese`, `daemon`, `dragonandcow`, `ghostbusters`, `kitty`, `moose`, `milk`, `stegosaurus`, `turkey`, `turtle`, `tux`.

#### 1.3.3. Exit the container

```bash
exit
```
(or `Ctrl+D`)

#### 1.3.4. Mount data into the container

By default, containers are fully isolated from the host filesystem. To grant access, **mount a volume**:

```bash
-v <outside_path>:<inside_path>
```

For example, mount the current directory as `/my_project`:

```bash
docker run --rm -it -v .:/my_project 'community.wave.seqera.io/library/cowpy:1.1.5--3db457ae1977a273' /bin/bash
```

Verify with `ls /my_project` — you should see your training files, including `data/greetings.csv`.

#### 1.3.5. Use the mounted data

```bash
cat /my_project/data/greetings.csv | cowpy -c turkey
```

This pipes the CSV contents through `cowpy`, though note it renders whole rows rather than just the greeting column — our Nextflow workflow will do a cleaner job of that. Exit when done:

```bash
exit
```

**Takeaway:** you know how to pull a container, run it one-off or interactively, and mount host data into it so tools inside can access real files — all without installing anything on your system.

---

## 2. Use containers in Nextflow

Nextflow has built-in support for running any process inside a container: it handles pulling the image, mounting data, and executing the process, all automatically. We'll add a `cowpy` step after `collectGreetings`.

### 2.1. Write a `cowpy` module

Create the module file:

```bash
touch modules/cowpy.nf
```

Populate it, modeled on the existing processes:

```groovy
#!/usr/bin/env nextflow

// Generate ASCII art with cowpy
process cowpy {

    input:
    path input_file
    val character

    output:
    path "cowpy-${input_file}"

    script:
    """
    cat ${input_file} | cowpy -c "${character}" > cowpy-${input_file}
    """
}
```

### 2.2. Add `cowpy` to the workflow

**a) Import the module** into `hello-containers.nf`:

```groovy
include { cowpy } from './modules/cowpy.nf'
```

**b) Call the process**, feeding it the collected output file plus a new `character` parameter:

```groovy
// generate ASCII art of the greetings with cowpy
cowpy(collectGreetings.out.outfile, params.character)
```

Recall `collectGreetings` emits two named outputs — `outfile` (the collected greetings) and `report` (the batch count) — so we specifically grab `outfile`.

**c) Declare the parameter with a default**, in the `params` block:

```groovy
params {
    input: Path = 'data/greetings.csv'
    batch: String = 'batch'
    character: String = 'turkey'
}
```

**d) Update `publish:`** to include the new output:

```groovy
publish:
first_output = sayHello.out
uppercased = convertToUpper.out
collected = collectGreetings.out.outfile
batch_report = collectGreetings.out.report
cowpy_art = cowpy.out
```

**e) Update the `output {}` block**, and take the opportunity to tidy up publishing destinations — pushing intermediate files into a subfolder and keeping only the final report and ASCII art at the top level:

```groovy
output {
    first_output {
        path 'hello_containers/intermediates'
        mode 'copy'
    }
    uppercased {
        path 'hello_containers/intermediates'
        mode 'copy'
    }
    collected {
        path 'hello_containers/intermediates'
        mode 'copy'
    }
    batch_report {
        path 'hello_containers'
        mode 'copy'
    }
    cowpy_art {
        path 'hello_containers'
        mode 'copy'
    }
}
```

**f) Run it:**

```bash
rm -r results/hello_containers/
nextflow run hello-containers.nf -resume
```

This fails with **exit status 127** — `cowpy: command not found`. That's expected: we haven't told Nextflow to actually use a container for this process yet.

### 2.3. Use a container to run the `cowpy` process

#### 2.3.1. Specify a container for `cowpy`

Add the `container` directive to `modules/cowpy.nf`:

```groovy
process cowpy {

    container 'community.wave.seqera.io/library/cowpy:1.1.5--3db457ae1977a273'

    input:
    path input_file
    val character

    output:
    path "cowpy-${input_file}"

    script:
    """
    cat ${input_file} | cowpy -c "${character}" > cowpy-${input_file}
    """
}
```

This tells Nextflow *which* image to use for this process — **if** container use is enabled.

#### 2.3.2. Enable Docker via `nextflow.config`

A `nextflow.config` file in the current directory is loaded automatically by Nextflow. The provided one currently disables Docker:

```groovy
docker.enabled = false
```

Flip it on:

```groovy
docker.enabled = true
```

> **Tip:** You can also enable Docker per-run from the CLI with `-with-docker <container>`, but that only lets you specify one container for the whole workflow. Setting `container` per-process in the module file (as above) is more modular, maintainable, and reproducible.

#### 2.3.3. Run with Docker enabled

```bash
nextflow run hello-containers.nf -resume
```

This time it succeeds — `cowpy` runs `1 of 1 ✔`, and the earlier cached steps (`sayHello`, `convertToUpper`, `collectGreetings`) are reused. The final output lands at `results/hello_containers/cowpy-COLLECTED-batch-output.txt`, containing ASCII art of a turkey reciting the greetings.

#### 2.3.4. Inspect how Nextflow launched the containerized task

Find the `cowpy` task's work subdirectory from the console log (e.g. `[98/656c6c]` → `work/98/656c6c...`), and open `.command.run`. Searching for `nxf_launch()` reveals something like:

```bash
nxf_launch() {
    docker run -i --cpu-shares 1024 -e "NXF_TASK_WORKDIR" \
      -v /workspaces/training/hello-nextflow/work:/workspaces/training/hello-nextflow/work \
      -w "$NXF_TASK_WORKDIR" --name $NXF_BOXID \
      community.wave.seqera.io/library/cowpy:1.1.5--3db457ae1977a273 \
      /bin/bash -ue /workspaces/training/hello-nextflow/work/98/656c6c.../.command.sh
}
```

Nextflow builds the `docker run` command itself: it mounts the task's work directory, sets the working directory inside the container, and runs the templated `.command.sh` script — all the manual steps from Section 1, fully automated.

**Takeaway:** you know how to use containers in Nextflow to run processes, without having to script any Docker commands yourself.

---

## Summary checklist

- [ ] I pulled a container image and ran it both one-off and interactively with Docker
- [ ] I mounted host data into a container using `-v`
- [ ] I wrote a `cowpy` process module with a `container` directive
- [ ] I connected the new process to the workflow and added its output to `publish:` / `output {}`
- [ ] I enabled Docker execution via `nextflow.config` (`docker.enabled = true`)
- [ ] I inspected `.command.run` to see how Nextflow launched the containerized task

## Key concepts covered

| Concept | Purpose |
|---|---|
| `docker pull` / `docker run` | Manually fetch and execute a container image |
| `-v <host>:<container>` | Mount host data into an isolated container |
| `container '<uri>'` | Declare which image a Nextflow process should run in |
| `docker.enabled = true` | Enable container execution globally via `nextflow.config` |
| `.command.run` / `nxf_launch()` | Where Nextflow records the exact container launch command |

## Next steps

Take a break — then move on to [Part 6: Hello Config](https://training.nextflow.io/3.6/hello_nextflow/06_hello_config/), the final part of the course, covering how to configure pipeline execution for your infrastructure.

## Resources

- [Community forum](https://community.seqera.io/c/training/39)
- [Nextflow Slack](https://www.nextflow.io/slack-invite.html)
- [GitHub issues](https://github.com/nextflow-io/training/issues)
- [Seqera AI assistant](https://seqera.io/ask-ai/chat-v2)
- [Nextflow Docs](https://nextflow.io/docs/latest/index.html)
- [Nextflow container documentation](https://nextflow.io/docs/latest/container.html)
