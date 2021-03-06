---
draft: false
title: "Syndesis CLI"
sidebar: "sidenav"
menu:
  sidenav:
    name: CLI
    weight: -400
    parent: Developer Docs
toc: true
weight: 10
---

## Buildtool "syndesis"

Syndesis uses a single tool for controlling various aspects of the build and developer related tasks.
This script can be found in `$SYNDESIS_DIR/tools/bin` and is called `syndesis`.
It requires bash and can be used on any Unix or macOS.

To have this script handy all the time it is recommended to either put this `bin/` directory into the path or add a symlink from `syndesis` into a directory which is already on your execution path.

The script can be used for various tasks which are selected by a so-called command_ which is the first argument to the script.

Just type `syndesis -h` to get an overview of the commands available:


### Usage Message
```
Usage: syndesis <command> [... options ...]

with the following commands

   build            Build Syndesis
   dev              Developer tools
   doc              Generate Syndesis Developer Handbook (SDH)
   minishift        Initialize and manage a Minishift developer environment
   system-test      Run system tests

"build" is the default command if no command is specified.
```


There are a handful of global options which can be used:

||||
| ------------- |--------| ------|
| `--help`      | `-h`   | Print usage information. If a command is given print out commands specific options. |
| `--rebase`    | `-r`   | Rebase your project directory before building to integrate upstream changes to your local repository. See below for details how this works. |
| `--verbose`   | &nbsp; | Set verbose mode, which is useful mostly only for debugging the script itself. |


### Rebase to Upstream

To easily rebase on changes which have been merged upstream to master, you can use the option `--rebase` (short: `-r`).
This command assumes that you have forked the Syndesis GitHub repositories and you have "origin" and "upstream" remotes like

```
$ git remote -v
origin    git@github.com:rhuss/syndesis.git (fetch)
origin    git@github.com:rhuss/syndesis.git (push)
upstream  https://github.com/syndesisio/syndesis (fetch)
upstream  https://github.com/syndesisio/syndesis (push)
```

With this in place, a `--rebase` performs the following steps:

* Refresh upstream remote: `git fetch upstream master`
* Try a `git rebase upstream/master` which rebases your current local working branch.
* If this fails because you have uncommitted work:
  - A `git stash` is performed
  - The rebase is retried and should succeed
  - `git stash pop` brings back your changes. Stashing can fail with conflicts which you would have to resolve on your own.


### Development Modes

The Syndesis application consists of a set of Docker images OpenShift resources descriptors for installing Syndesis.
For development, [minishift](https://www.openshift.org/minishift/) is used, and most of the commands assume that you have minishift installed locally and executable directly from your path.
Minishift can be downloaded and installed from https://github.com/minishift/minishift/releases.

For development OpenShift S2I builds and image streams are used.
This mode works also when running with a real OpenShift cluster and is not restricted to Minishift usage.
The advantage is that a build automatically triggers a redeployment, so you don't have to kill any pods manually.


### Commands

All other options are specific to each command.
You get a list of those options with `syndesis <cmd> -h`.
These options are described in detail in the next sections.

The following commands are available:

|Command      | Description|
| ----------- | -----------|
| build       | Used for building Syndesis and its various modules   |
| ui          | Start the UI for local development _(not implemented yet)_ |
| minishift   | Start and install Syndesis on Minishift |
| install     | Install Syndesis in a running cluster (other than Minishift) |
| system-test | Run System test against an OpenShift cluster |
| dev         | Utility commands useful during development |
| doc         | Generating and publish this documentation |
| release     | Release Syndesis' Maven and NPM artifacts _(not implemented yet)_ |

If no command is given, `build` is the default.
Remember a command must be the first argument, but there are additional possibilities to specify commands:

* You can use the form `--command` anywhere as an option, too. E.g. using  `--minishift` is the same as specifying "minishift" as the first argument.
* A `--mode command` can be used, too (e.g. `--mode sytem-test`)

The next sections describe the commands in detail.
To add a new command, just drop a script file into `$SYNDESIS_DIR/tools/bin/commands` directory with following structure:

```
#!/bin/bash

yourscriptname::description() {
    echo "Describe the command"
}

yourscriptname::usage() {
    cat <<EOT
    Describe the usage of the command
EOT
}

yourscriptname::run() {
    Do your stuff
}
```

## syndesis build

The primary goal of `syndesis build` is to build Syndesis.

It is mainly a wrapper around Maven and Go but adds some convenience
functionality for common developer workflows.

### Usage

    Usage: syndesis build [... options ...]
    
    Options for build:
        -b  --backend                  Build only backend modules (core, extension, integration,
                                       connectors, server, meta)
            --all-images               Build all modules with images: ui, server, meta, s2i, operator
            --app-images               Build only application modules with Docker images
                                       (ui, server, meta, s2i)
                                       and create images
            --infra-images             Build only infrastructure modules with Docker images
                                       (operator) and create images
        -m  --module <m1>,<m2>, ..     Build modules
                                       Modules: ui, server, connector, s2i, meta, integration,
                                       extension, common, operator
        -d  --dependencies             Build also all project the specified module depends on
            --skip-tests               Skip unit and system test execution
            --skip-checks              Disable all checks
        -f  --flash                    Skip checks and tests execution (fastest mode)
        -i  --image                    Build Docker images, too
                                       (for those modules creating images)
            --docker                   Use a plain Docker build for creating images.
                                       Used by CI for pushing to Docker Hub
        -p  --project <project>        Specifies the project/namespace to create images in
        -e  --ensure                   If building the operator, run 'dep ensure' before building.
                                       Otherwise ignored.
        -l  --local                    If building the operator, use a locally installed go
                                       Otherwise run the Go build from a container
                                       (either local or Minishift's Docker daemon)
            --clean-cache              Used for the operator build to remove the local dependency cache.
        -c  --clean                    Run clean builds (mvn clean)
            --resume <m1>              Resume build from maven module (mvn -rf m1)
            --batch-mode               Run mvn in batch mode
            --camel-snapshot <version> Run a build with a specific Camel snapshot.
                                       If no argument is given the
                                       environment variable CAMEL_SNAPSHOT_VERSION is used or an error thrown.

### Modules

A plain `build` command without any options performs a plain `mvn
install` for all Java and UI modules. Plus it also builds the
infrastructure operator via Go (see [syndesis-build-operator](#infrastructure-operator)
for details)

This compiles all Java and Javascript artifacts and also runs all tests
and code checks for the application modules.

You restrict the build to certain modules, which are divided into two
categories: *application modules* which are the modules building up
Syndesis. And *infrastructure modules* which help in managing the
application itself.

You can individually select specific modules by using the `--module`
(short: `-m`) option with a comma-separated list of modules, but there
are also option to combine modules

The following modules are
available:

| Module          | Description                                                                            | `--all-images` | `--backend` | `--app-images` | `--infra-images` |
| --------------- | -------------------------------------------------------------------------------------- | -------------- | ----------- | -------------- | ---------------- |
| **server**      | Main backend providing a REST API for the user interface                               | ✔︎             | ✔︎          | ✔︎             |                  |
| **ui**          | The SPA user interface application                                                     | ✔︎             |             | ✔︎             |                  |
| **meta**        | Meta data and verifier used for verifying connections and providing connector metadata | ✔︎             | ✔︎          | ✔︎             |                  |
| **connector**   | All connectors used by Syndesis out of the box                                         |                | ✔︎          |                |                  |
| **integration** | Support libraries for running integrations                                             |                | ✔︎          |                |                  |
| **extension**   | Tools for developing and running Syndesis custom extensions                            |                | ✔︎          |                |                  |
| **common**      | Common modules containing common code                                                  |                | ✔︎          |                |                  |
| **s2i**         | S2I base image used for building the runtime images                                    | ✔︎             |             | ✔︎             |                  |
| **operator**    | Infrastructure operator for managing the application                                   | ✔︎             |             |                | ✔︎               |

All option ending with `-images` will also build the corresponding
Docker image.

When you build individual modules you you can provide the option
`--image` (short: `-i`) to create also the Docker image in the build,
when the module is associated with a Docker image.

By default images are build via S2I against a running Minishift. This is
the recommended way for developing as this automatically will trigger a
redeployment after the build. However, for certain scenarios like when
used in a CI system or when doing the release, the image creation can be
done against are Docker daemon when the `--docker` is given. For this to
work you must have access to Docker daemon, which you can verify with
`docker ps`.

When the option `--dependencies` (short: `-d`) is given in addition to
`--modules`, also all Maven modules which the specified modules depend
on are build, too.

### Tuning

By default, all checks like license or code quality checks are
performed. Also, all unit and local integration tests are run. A full
build eats up quite some time, but you should always run at full blast
before submitting a pull request.

However, to speed up the turnaround, several speed-up options are
available. The following table shows these options, and also how long a
full clean build over all modules takes: (but without building
images)

| Option          | Description                                                                                       | Time |
| --------------- | ------------------------------------------------------------------------------------------------- | ---- |
| *none*          | Default mode with all checks and tests                                                            |      |
| `--skip-tests`  | Skip all unit and local integration tests                                                         |      |
| `--skip-checks` | Skip sanity checks like for correct license headers and                                           |      |
| `--flash`       | Fastest mode with skipping all checks and tests and with even some other aggressive optimizations |      |

### Infrastructure Operator

`syndesis build` can also build the infrastructure operator, which is a
golang program.

You can build the operator by running `syndesis build -m operator` or as
part of a module collection like `--all-images` or `--infra-images`

There are three modes, how the operator can be created:

  - Running you go compiler locally

  - Compiling in a local Docker daemon which allows volume mounts with
    the localhost

  - Compiling in Minishift Docker daemon, which was made accessible via
    `eval $(minishift docker-env)`

See below for the details.

**Load dependencies**

In any case, before you compile first you should use the option
`--ensure` (short: `-e`) to setup the dependency tree. This will
download all source dependency and cache them locally. To get rid of
this cache, use the option `--clean-cache`. This might be necessary when
`go dep` 's cache gets into a weird state.

If you compile for the first time, then `--ensure` will be added
automatically.

**Compiling locally**

This is the fastest way for compiling the operator. Use the option
`--local` (short: `-l`) for selecting the local compile mode.

It is also the recommended way when you are working on the operator. You
project setup needs to fit however: The main project directory must be
reachable as `$GOPATH/src/github.com/syndesisio/syndesis`. You can
either move your project directory to this location or work with a
symlink:

``` shell
cd ~/Development/syndesis
mkdir -p $GOPATH/src/github.com/syndesisio
cd ..
mv syndesis $GOPATH/src/github.com/syndesisio/
ln -s $GOPATH/src/github.com/syndesisio/ syndesis
```

By default this compiles into for your native architecture (amd64,
darwin). When you use this mode with `--image` (short: `-i`) on macOS
then go will be used as cross compiler so that the generated bimary can
be used in a Linux image.

**Compiling with a local Docker daemon**

This is the default mode and is used also when doing the release. It use
a builder image `syndesis/godev` which is created from the `tools/image`
directory and fetched from Docker Hub.

For this mode to work your Docker daemon must support volume mounts to
the system from where you are calling `syndesis`. This is the case on
Linux for locally installed Docker daemon and for Mac with *Docker for
Mac*. It is **not** the case for Minishift which runs in a disconnected
VM. But see below how you still can use Minishift for building.

`dep ensure` and `go build` will be run from this `syndesis/godev`
image, but with your local directory mounted into the container so that
the fetched dependencies can be cached in the local directories
`dep-cache` and `vendor` so that the can be reused for the next run.
Also the binary will be stored in your local directory, but this will
alway be a Linux (`amd64`) binary.

**Compiling with Minishift**

As Minishift is running in a remote VM you cannot bind a volume to your
localhost. Therefor when running in Minishift mode the source code will
be rsynced to a directory in the Minishift VM (`rsync` will be installed
in the Minishift VM on the first run).

The build with `syndesis/godev` will then be started with a volume mount
to the copied directory in the VM. After the build the generated binary
is copied back with `rsync` to your local directory.

The Minishift mode is automatically detected and selected if
`DOCKER_CERTS` contains a path to `.minishift`. You enable the Minishift
Docker daemon for your local CLI with `eval $(minishift docker-env)`

Some simple benchmark reveals the following timings (in minutes)
:

| Context               | Local | Local Docker (macOS) | Minishift Docker |
| --------------------- | ----- | -------------------- | ---------------- |
| Cold (no build cache) | 0:40  | 10:09                | 1:36             |
| Hot (with dep cache)  | 0:08  | 2:47                 | 0:13             |

The fastest mode is obviously the local mode, followed surprisingly by
Minishift. The local mode is probably slow because of how the macOS
daemon mounts ts volumes (which can probably be optimized)

### Examples

Some common usage examples for `syndesis build` are

``` shell
# Build all images (app and infrastructure) with S2I
syndesis build --all-images

# Create all application images and re-deploy Syndesis in the
# Openshift cluster, but do it as fast as possible. Don't build
# any golang code
syndesis build --app-images --flash

# Create the infrastructure operator by running go locally
# and calling dep ensure before
syndesis build -m operator --local --ensure

# Use a Camel snapshot for a clean build, build all modules
syndesis build --clean --camel-snapshot 2.21.0-SNAPSHOT
```



## syndesis ui

<div class="alert alert-info admonition" role="alert">
  <i class="fa note"></i> "syndesis ui" is not implemented yet.
</div>

## syndesis minishift

With `syndesis minishift` you can adequately manage a
[minishift](https://www.openshift.org/minishift/) installation for
hosting Syndesis. This command is especially useful for a simple and
self-contained development workflow.

`syndesis minishift` requires that you have a current minishift in your
path. You can download it directly from
[GitHub](https://github.com/minishift/minishift/releases).

### Usage

    --install                 Install templates to a running Minishift.
    -p  --project             Install into this project.
                              Delete this project if it already exists.
                              By default, install into the current project (without deleting)
    -c  --context             Install into this context. Default is 'minishift'
    --reset                   Reset and initialize the minishift installation by
                              'minishift delete && minishift start'.
    --full-reset              Full reset and initialie by
                              'minishift stop && rm -rf ~/.minishift/* && minishift start'
    --memory <mem>            How much memory to use when doing a reset. Default: 4912
    --cpus <nr cpus>          How many CPUs to use when doing a reset. Default: 2
    --disk-size <size>        How many disk space to use when doing a reset. Default: 20GB
    --vm-driver <driver>      Which virtual machine driver to use (depends on OS)
    --show-logs               Show minishift logs during startup
    --openshift-version <ver> Set OpenShift version to use when reseting (default: v3.9.0)
    --tag <tag>               Syndesis version/tag to install. If not given, then the latest
                              version from master is installed
    --local                   Use the local resource files instead of fetching them from GitHub
    -o  --open                Open Syndesis in the browser
    -y  --yes                 Assume 'yes' automatically when asking for deleting
                              a given project.
    --memory-server <mem>     Memory limit to set for syndesis-server. Specify as "800Mi"
    --memory-meta <mem>       Memory limit to set for syndesis-meta. Specify as "512Mi"
    --test-support            Allow test support endpoint for syndesis-server
    --man                     Open HTML documentation in the Syndesis Developer Handbook

### Installing Syndesis

You can easily install Syndesis with the option `--install`. This option
triggers the creation of all relevant OpenShift resources objects in the
currently connected OpenShift project.

If you want to use a different project, then use `--project` (short:
`-p`) to specify this project.

<div class="alert alert-info admonition" role="alert">
  <i class="fa warning"></i> Any existing project will be deleted first when specified with `--project`. This option is also an easy and quick way to recreate a Syndesis installation.
</div>

### Resetting Minishift

The quickest way to get a fresh Syndesis setup is to use `--project`
which will install Syndesis into a clean, new project.

However, you can also recreate the whole Minishift installation with
`--reset`. This will delete the Minishift VM (`minishift delete`) and
create a new one (`minishift start`). It doesn’t harm if the Minishift
VM does not exist so that you can use `--reset` also on a fresh
Minishift installation.

If you want to get a real clean installation use `--full-reset` which
deletes the `~/.minishift` directory which holds downloaded artifacts
like the ISO image for the Minishift VM. Using `--full-reset` forces
Minishift to re-download all those files.

There are several options which influence the re-creation of the
VM:

| Option                | Description                                                                                                      | Default |
| --------------------- | ---------------------------------------------------------------------------------------------------------------- | ------- |
| `--memory`            | Memory to use for the Minishift VM.                                                                              | 4 GB    |
| `--cpus`              | Number of CPUs used for the Minishift VM.                                                                        | 2       |
| `--disk-size`         | Disk space used for Minishift.                                                                                   | 20 GB   |
| `--show-logs`         | Whether to show OpenShift logs during startup.                                                                   | false   |
| `--vm-driver`         | Which virtual machine driver to use. For OS X this can be 'virtualbox', 'xhyve' or 'vmwarefusion' (if installed). |         |
| `--openshift-version` | OpenShift version to use                                                                                         | 3.7.1   |

### Example

This short example performs the following actions:

  - Stops and deletes a running Minishift VM (if existent)

  - Removes `~/.minishift` (if existent)

  - Install Syndesis in OpenShift modes (S2I builds & image streams) in
    project `syndesis`

  - Open Syndesis UI in the default browser

<!-- end list -->

    # Complete fresh installation in project "syndesis"
    syndesis minishift --full-reset --install --project syndesis
    
    # Open Syndesis in default browser
    syndesis minishift -o


## syndesis system-test

The `system-test` command is for running a full blown system test of Syndesis by installing it in one of multiple projects which are managed in a pool.


### Usage

```
Usage: syndesis system-test [... options ...]

Options for system-test:
    --project <project>       The test project to use
    --token <token>           Token for connecting to the server
    --server <url>            OpenShift server url to use for the tests.
                              If not given, use the currently connected server
    --pool <project>          If no project is given, use a pooling mechanism.
                              This pool has to be created before with --create-pool
    --test-id <id>            Id to identify the test run
    --create-pool <prefix>    Create project pool for system-tests with
                              all projects with the given prefix
    --list-pool               Show all locks for the pool
    --release-project <t-id>  Release project for given test id (or all if no test id is given)
```

### How it works

<div class="alert alert-info admonition" role="alert">
  <i class="fa note"></i> This section needs a more detailed explanation how the systems tests are working in detail.
</div>



## syndesis dev

Dev commands are useful helpers for developing Syndesis

### Usage

    Usage: syndesis dev [... options ...]
    
    Options for dev:
        --debug <name>            Setup a port forwarding to <name> pod (default: server)

This command enable port-forwarding of port 5005 from a specific pod (by
default: "server") to port 5005 on the localhost. You then can point
your Java IDE to port 5005 on localhost for connecting for remote
debugging. As argument to `--debug` "server", "meta" and "atlasmap" can
be used, which are our Java based services.


## syndesis doc

This command is used to generated and manage this documentation which
you are currently reading.

### Usage

    Usage: syndesis doc [... options ...]
    
    Options for doc:
    -d  --directory <dir>         Top-level dir holding doc source and output directory. Default: "doc/sdh"
    -i  --input <file>            Input file to use. Default: "index.adoc"
        --out <dir>               Directory to generate files (default: "output")
        --html                    Generate HTML pages
        --pdf <out>               Generate PDF and write it to <out> (default: "sdh.pdf")
        --epub <out>              Generate Epub and write it to <out> (default: "sdh.epub")
        --gh-pages <msg>          Create everything into the gh-pages branch and commit with <msg>
    -o  --open                    Open HTML documentation in default browser after doc generation
    -l  --local                   Use locally installed commands instead of Docker image

`syndesis doc` uses by default a Docker container to create the
documentation. Therefore a Docker daemon must be accessible, and it must
allow [bind volume
mounts](https://docs.docker.com/engine/admin/volumes/bind-mounts/) to
the local `doc/sdh` directory. Bind mounts are possible for Linux Docker
daemons and for "Docker for Mac". But it is not the case for the
Minishift exposed Docker daemon, as this daemon is running isolated in a
VM.

You can always run `asciidoctor` locally with the `--local` (short:
`-l`) options. See the [Asciidoctor
manual](https://asciidoctor.org/#installation) for more information how
to install Asciidoctor.

### Output

Input and output has the same defaults but can be changed with the
following
options

|                  |             |                                                                                          |
| ---------------- | ----------- | ---------------------------------------------------------------------------------------- |
| `--input <file>` | `-i <file>` | Input document in Asciidoc format which is by default `$SYNDESIS_DIR/doc/sdh/index.adoc` |
| `--output <dir>` | `-o <dir>`  | Output directory, default is `$SYNDESIS_DIR/doc/sdh/output`                              |

By default `syndesis doc` creates the documentation in HTML format, but
more formats are
supported:

|                 |                                                                                                                                                                                    |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--html`        | Create HTML documentation in the output directory                                                                                                                                  |
| `--pdf <name>`  | Create PDF documentation. `<name>` is optional, but when given, then this specifies the file name generated in the output directory. By default `sdh.pdf` is used.                 |
| `--epub <name>` | Create the documentation in Epub format. `<name>` is optional, but when given, then this specifies the file name generated in the output directory. By default `sdh.epub` is used. |


## syndesis release

Use `syndesis release` for performing a release of Syndesis. A Syndesis
release consists of:

  - Maven artifacts of the backend and runtime services

  - Docker images pushed to Docker Hub

  - A set of OpenShift templates referencing these images

This chapter describes how you can efficiently perform a release and how
to troubleshoot if something goes wrong. This documentation might also
be interesting to you even when you do not perform a release on your
own, as it might help you to understand how the various Syndesis
artifacts fit together.

### Usage

    Perform a release
    
    This command performs a release but does no automated rollback when something fails.
    Please refer to the manual at https://docs.syndesis.io for more details and what needs
    to be cleaned up in case of a failure.
    
    Usage: syndesis release [... options ...]
    
    Options for release:
    -n  --dry-run                 Dry run, which performs the whole build but does no tagging,
                                  artifact upload or pushing Docker images
        --release-version <ver>   Version to release (e.g. "1.2.1"). This is a mandatory argument.
        --snapshot-release        Snapshot release which can be created on a daily basis.
                                  A timestamped version will be created automatically, and no Maven artifacts
                                  are pushed to maven central. No moving tag will be moved, too.
        --dev-version <version>   Next development version. If not given, set to
                                  "<major>.<minor>-SNAPSHOT" as calculated from
                                  --release-version (e.g. "1.2-SNAPSHOT")
        --settings <file>         Path to a custom settings.xml to use for the release.
                                  This file must contain all the credentials to be used for Sonatype.
                                  By default ~/.m2/settings.xml is used.
        --local-maven-repo <dir>  Local dir for holding the local Maven repo cache.
                                  If not given, then a new
                                  temporary directory will be used
                                  (and removed after the release)
        --docker-user <user>      Docker user for Docker Hub
        --docker-password <pwd>   Docker password for Docker Hub
        --no-git-push             Don't push the release tag (and symbolic major.minor tag)
        --git-remote              Name of the git remote to push to. If not given, its trying to be pushed
                                  to the git remote to which the currently checked out branch is attached to.
                                  Works only when on a branch, not when checked out directly.
    
        --log <log-file>          Write full log to <log-file>, only print progress to screen
        --man                     Open HTML documentation in the Syndesis Developer Handbook

A lot of output is produced during a release. If you are only interested
to see the major steps only, then you can use `--log` to redirect the
output to a specific log file. This log file will contain all output
(and if you add `--verbose` you see even more output), but on the
console you will only see the current step that is actually performed.

**Example**

An example run for a dry run for `1.3.1` release on the current branch
look like:

``` bash
./tools/bin/syndesis release \           1
    --release-version 1.3.1 \            2
    --local-maven-repo /tmp/clean-repo \ 3
    --log /tmp/build.log \               4
    --dry-run                            5
```

  1. Always run `syndesis` from the repo and branch you want to release.

  2. The release version is mandatory and must be in the format
    `<major>.<minor>.<patch>`.

  3. Use a clean local Maven repository to avoid side effects

  4. Redirect the full output to `/tmp/build.log` but still print the
    main steps to the console.

  5. Make only a dry run, without pushing any artifacts out nor checking
    in any changed files.

### Preparations

To perform a release, certain preconditions need to be given.

First of all, you need to have access to the various systems to which
release artifacts are uploaded:

  - You need to be logged in to [Docker Hub](https://hub.docker.com/)
    and your account needs to have write access to the
    [syndesis](https://hub.docker.com/u/syndesis/) Docker Hub
    organisation.

  - You have `gpg` to have installed and set up a gpg-agent for being
    able to sign Maven artifacts during deployment in a non-interactive
    mode.

  - You need to have access to the "syndesis" account on
    ([oss.sonatype.org](https://oss.sonatype.org/) for being able to
    publish Maven artifacts. This credential needs to be added to either
    your `~/.m2/settings.xml` or you can use an settings file with the
    `--settings-xml` option. The credential needs to be added to the
    server with the id `oss-sonatype-staging`.

You have to perform the release from a locally checked out Syndesis
repository, which can be either checkout from a branch like `1.2.x` for
a patch release or directly from `master`. It is highly recommended to
run the release directly from a freshly checked out directory:

``` bash
# Go to a temporary directory
cd /tmp

# Clone repository afresh, but only use the last history entry (--depth=1)
git clone --depth=1 https://github.com/syndesisio/syndesis.git

# Jump into the directory
cd syndesis

# Switch to the target branch if needed (or stay on master)
git fetch --depth 1 origin 1.2.x:1.2.x
git checkout 1.2.x

# Call Syndesis from the checked out clone
./tools/bin/syndesis release --release-version 1.2.8 .....

# Push to origin after a successful release.
# This automatically done if --no-git-push is given
git push 1.2.8
git push -f 1.2

# Remove the temporary clone again
cd ..
rm -rf syndesis
```

Please note that you should always call `syndesis` out of the branch for
which the release is for. If there is an issue due to bugs in the
release script itself, please fix them on the branch with the usual
developer process (i.e. opening a PR request). `syndesis release` must
always work for the branch where this script is, too.

### Release steps

A release consist of several different steps, which can be grouped into
two groups:

  - **Build steps** are performed to build the release and create the
    artifacts. Also during the build Maven artifacts are uploaded to the
    staging area for publishing to Maven central

  - **Persist steps** are then used for releasing objects, pushing
    Docker images to Docker Hub, committing and tagging in Git (but only
    when the build steps have been performed successfully).

**Build steps**

  - Check whether the current local Git clone is *clean*, i.e. that is
    does not have any modified files. The script will abort if this is
    the case.

  - Update the versions of all `pom.xml` files below `app/` to the
    version given with `--release-version`. If no `--release-version` is
    given, then the script aborts.

  - Run an `mvn clean install` to verify that the build is not broken
    and all tests succeed.

  - Re-generate the OpenShift templates in `install` so that the image
    streams included in these templates refer to Docker images with the
    new version.

  - Now run an `mvn -Prelease clean deploy` to deploy all artifacts to a
    new staging repository on oss.sonatype.org, the platform for release
    artifacts on Maven central. The staging repository on this Sonatype
    Nexus is validated and closed.

  - If `--docker-user` and `--docker-password` is given, then a `docker
    login` is performed. Otherwise, it is assumed that the user is
    already logged in.

  - The Docker images are created with `mvn -Prelease,image package` in
    the `server`, `meta`, `ui` and `s2i` modules.

If the option `--dry-run` (short: `-n`) is provided, the script drops
the staging repository at Sonatype and stops. You should examine the
generated files and before starting a real build, reset the repository
(`git reset --hard`).

The builds are using a clean local Maven repository, which otherwise is
usually taken from `~/.m2/repository`. This new local cache should
ensure that we have a completely fresh build without interference from
previous builds store in the local Maven cache in the home directory.
You can provide such a directory with `--local-maven-repo` which will be
taken directly (so it’s good if you have to perform multiple runs like
with `--dry-run`). If not provided, a new temporary directory is created
and also *deleted* after the release run.

**Persist Steps**

  - Push Docker images to Docker Hub. In addition to the images that
    carry the full release version as the tag, also a tag for the *minor
    version* is attached and pushed. E.g. when the release version is
    `1.2.8`, then the minor version is `1.2`. If this minor version tag
    already exists on Docker Hub, its moved to the newly created
    version.

  - The staging repository on Sonatype is released. It will take a bit,
    but the artifact should then be downloadable from [Maven
    central](https://search.maven.org/) soon after.

  - Commit all modified local files to the local Git repo.

  - Create a Git tag for the release version (e.g. `git tag 1.2.8`).

The next steps are for creating templates for the minor version:

  - In `install` create new templates which contain image streams that
    reference images with the minor version (e.g.
    `syndesis/syndesis-server:1.3` for a release version of 1.3.8).

  - Commit those generated templates

  - Tag it with the minor version (e.g. `1.2`), overwriting an already
    existing minor version tag

Next, we are switching back to the next development version of the
pom.xml files. This version can be given with `--dev-version`, but by
default, it is calculated automatically as `<minor.version>-SNAPSHOT`
(e.g. `1.2-SNAPSHOT`). This new version is then committed to the local
git repository.

Finally, the tags just created on the local Git repo is pushed to the
remote repository. You can omit this with the option `--no-git-push`. If
to so, the last step can also be performed manually afterwards with:

``` bash
git push 1.2.8   
git push -f 1.2   1
```

  1. Using `-f` as the minor tag needs to be moved.

Please be careful to **not** push the master branch upstream (i.e. do
**not** a plain `git push`). We only want to have the tag with all the
release preparation steps, not on the branch so that pull requests can
be still be easily rebased with out conflict because of the temporary
version changes.

### Minor Version Templates

What is now the thing with this *minor version*? Why is the needed and
how does it work?

Syndesis follows a [semantic versioning](https://semver.org/) approach.
So, patch level releases (i.e. all releases which only change the last
digit in 1.2.8) are fully compatible with all other patch level
versions. In order to allow easy bug fix upgrades, we also create a tag
which contains only the version parts up to the minor version (e.g.
1.2). These tags **always** points to the latest full version of its
minor version. If, e.g. 1.2.8 is the latest 1.2.x version, then the tag
1.2 point to this 1.2.8 version. Corresponding to these Docker image
variants, there exist two OpenShift templates variants:

  - One set of templates directly references the Docker images which its
    full version, e.g. `syndesis/syndesis-ui:1.2.8`. Applying such a
    template will keep your application at precisely this patch-level.
    You would have to update your templates and recreate your
    applications if you want to upgrade.

  - The other set of templates references images only via its minor
    version, e.g. `syndesis/syndesis-ui:1.2`. Using these templates has
    the advantage that application created from these templates
    automatically benefit from patch releases. The templates contain an
    image change trigger which will redeploy the application if the
    images change. So when we release the next patch level release,
    moving the minor version tag to this patch level release, then the
    application gets automatically redeployed, and it will pick up the
    new image.

These two sets of templates can be reached directly from GitHub as the
git tags correspond to the Docker tags (i.e. a `1.2.8` tag and a `1.2`
tag which will be moved forward).

### Snapshot Release

With the option `--snapshot-release` a lightweight snapshot release for
the images and templates can be created. The tag/version is calculated
automatically by picking up the latest release number (e.g. 1.3.5),
increasing the patch-level by one and adding a daily timestamp (e.g.
1.3.6-20180419). According to [Semantic
Versioning 2.0](https://semver.org/) this is considered to be a version
larger than 1.3.5 but [smaller](https://semver.org/#spec-item-11) than
1.3.6.

This tag can be referenced to in `syndesis install` and `syndesis
minishift`.

In detail, a snapshot release differs from a normal release as it:

  - …​ doesn’t release artifacts on Maven central, but pushes Docker
    images and creates a Git tag for referencing the proper templates.

  - …​ skips all checks and tests when building to maximise the
    likelihood that the release succeeds. The rationale here is to
    better have untested daily snapshot release than no snapshot release
    because of test failure (which in many cases are not because of
    errors, but of failure in the infrastructure)

  - …​ force pushes the snapshot tag on GitHub so that multiple releases
    per day are allowed

**Example**

``` bash
syndesis release \
     --snapshot-release \                   1
     --local-maven-repo /tmp/clean-repo \   2
     --git-remote origin \                  3
     --docker-user "${DOCKER_USER}" \       4
     --docker-password "${DOCKER_PASSWORD}"
```

  1. Enable snapshot release with a version in the format 1.3.5-20180419

  2. Point to an empty repository to avoid side effects when building

  3. Push to the origin repository

  4. Docker credentials required for pushing to Docker Hub

A daily Jenkins job with this configuration run on
<https://ci.fabric8.io> for creating a daily snapshots.

### Troubleshooting

When you run the `syndesis release` command and when it should not
succeed, you might have to perform some cleanup steps yourself (there is
now automatic rollback). However, care has been taken to move all
persistent changes to the end of the release flow, so if something
breaks early, you only need to clean up locally. If the process fails
before the step *=== Pushing Docker images* you only need to:

  - Reset your local git repo with `git reset --hard`

  - Potentially remove the create staging repository on
    `http://oss.sonatype.org/` (but it doesn’t harm if it is not cleaned
    up immediately).

After pushing the Docker images, it should be improbable that things go
wrong. But these things should take care of if this should be the case:

  - Remove Docker Hub tags for the pushed images, which is best done on
    the Docker Hub Web UI

  - Revert your local git commits to the point before the release. If
    you did this on a fresh checked out repo (as recommended), you just
    could delete the whole clone.



## syndesis install

With `syndesis install` you can install Syndesis to an arbitrary
OpenShift cluster. If you want to install to
[Minishift](https://www.openshift.org/minishift/) the [syndesis
minishift](#syndesis-minishift) command is recommended as it supports
some additional features specific to
    Minishift.

### Usage

    -s  --setup               Install CRDs clusterwide. Use --grant if you want a specific user to be
                              able to install Syndesis. You have to run this option once as
                              cluster admin.
    -u  --grant <user>        Add permissions for the given user so that user can install
                              the operator in her projects. Must be run as cluster admin.
        --cluster             Add the permission for all projects in the cluster
                              (only when used together with --grant)
    -p  --project             Install into this project. Delete this project
                              if it already exists. By default, install into the current project
                              (without deleting)
        --operator-only       Only install the operator but no resource
        --route               Route to use. If not given, the route is trying to be detected
                              from the currently connected cluster.
        --console             The URL to the OpenShift console
        --tag <tag>           Syndesis version/tag to install. If not given, then the latest
                              version from master is installed
        --dev                 Prepare for development of Syndesis so that S2I builds of
                              Syndesis images are picked up properly (implies --watch)
        --force               Override an existing "Syndesis" if present
    -w  --watch               Wait until cluster is up
        --local               install from local Git repo when using. By default the
                              resource descriptor is downloaded from GitHub remotely.
    -o  --open                Open Syndesis in browser when installation is ready (implies --watch)
    -y  --yes                 Assume 'yes' automatically when asking for deleting
                              a given project.
        --memory-server <mem> Memory limit to set for syndesis-server. Specify as "800Mi"
        --memory-meta <mem>   Memory limit to set for syndesis-meta. Specify as "512Mi"
        --test-support        Allow test support endpoint for syndesis-server
    
    You have to run --setup --grant <user> as a cluster-admin before you can
    install Fuse Online as a user.

The deployment happens to the currently connected OpenShift cluster. So
it’s mandatory that you have logged into the cluster with `oc login`
before. You can check the status with `oc status`.

The installation process consists of two steps:

  - An initial setup which has to be performed as *cluster admin* which
    is a one-off action which needs to be done only once.

  - Installation of Syndesis into a specific project by the *app admin*,
    a regular user of OpenShiftm which can be performed as many times as
    required.

### Initial Setup

In the initial setup, you have to register the custom resource
definition (CRD) to allow to deploy `Syndesis` resources. This step has
to be performed by the admin.

Also, if you want to allow an OpenShift user to install Syndesis on her
own, then you have to grant specific permissions to her.

To perform this setup step, which needs to be performed only once per
cluster, you have to run `syndesis` **while being connected as a cluster
admin**. For `minishift` use `oc login -u system:admin` if you have the
`admin-user` addon enabled.

``` bash
syndesis install --setup
```

This will install only the CRD. In addition to grant a user *developer*
the proper permission to create a `Syndesis` resource, you should add
`--grant <user>`:

``` bash
syndesis install --setup --grant <user>
```

This call adds permissions to read and write `Syndesis` resource objects
for the current project. If you would instead want to allow the user
managing `Syndesis` resources in the whole cluster, you should add a
`--cluster`. This cluster-wide access is especially required when you
plan to use the `--project` option to use a new project or recreate the
existing one, as in this case the role association to this project gets
lost.

### Installing Syndesis

After the CRDs a registered, you can easily install Syndesis directly
with

``` bash
syndesis install
```

Depending on whether you have granted the current user access this step
has to be done either as admin or as a regular user.

**Example for Minishift**

``` bash
# Enable the admin user on Minishift
minishift addons enable admin-user

# Create a minishift instance
minishift start --memory 4192

# Switch to admin
oc login -u system:admin

# Register CRD and grant permissions to "developer"
syndesis --setup --grant developer --cluster

# Switch to account developer
oc login -u developer

# Install Syndesis
syndesis install
```

A route name can be given with `--route`. This step can be omitted as
the operator can autodetect the route. If you provide the route
manually, you need to check your OpenShift installation. Typically the
route name is the name of your OpenShift project followed by the
cluster’s hostname. E.g. a route `--route
proj186023.6a63.fuse-ignite.openshiftapps.com` is specific to the Fuse
Ignite test cluster `6a63.fuse-ignite.openshiftapps.com` and for the
project `proj186023`. However, you don’t have to provide the route name.

If you want to have a link to the OpenShift console to read the Pod
logs, you have to add the `--console` option with the full URL to the
console. Unfortunately, this cannot be autodetected. If not given, no
link appears.

By default, this commands installs Syndesis in the currently connected
project, but you can specify an alternative project with `--project
<project>`. If this project already exists, it gets deleted
unconditionally before the deployment, so be careful when using this
option. By default, you are asked whether you want to delete the project
for recreation. You can switch off the security question with the option
`--yes` (short: `-y`).

<div class="alert alert-info admonition" role="alert">
  <i class="fa warning"></i> Don’t use `syndesis install --project $(oc project -q) --yes`. You’ll shoot yourself into the foot. Ask the author if you want to know more details.
</div>

If you want to wait until everything is running (including fetching of
the Docker images), you can specify `--watch` (short: `-w`) which blocks
the script until everything is set up.

You can also automatically open Syndesis in the browser after the
installation with `--open` (short: `-o`)

### Development Mode

As with [syndesis minishift](#syndesis-minishift) you can also use this
command to set up a development platform for Syndesis. *Development
platform* here means that you can create Docker images on your own with
[syndesis build](#syndesis-build) and can use them with an automatic
redeployment after the build.

You can switch on this mode with the option `--dev`. When the operator
has deployed the application, the application imagestreams refer to
Docker images pushed to Docker Hub. To change the imagestream references
to the images built with `syndesis build --all-images`, these
imagestreams needs to be patched after the initial images have been
fetched from Docker Hub. If you use the `--dev` option, then this update
is done automatically.

### Selecting the Version

With the option `--tag` you can select a specific version of Syndesis to
install. By default, the currently checked out checked out branch is
used.

**Example**

``` bash
syndesis install --route syndesis.192.168.64.12.nip.io --tag 1.4
```

This example installs the latest Syndesis version of the 1.4 branch to
the local cluster.

You can see a list of available tags with `git tag`. Tags prefixed with
`fuse-ignite` are suited for the Fuse Online cluster as those templates
do not contain images streams themselves but refer to the image streams
installed on this cluster.

### Quick Installation

<div class="alert alert-info admonition" role="alert">
  <i class="fa warning"></i> The following scripts are not yet updated and probably don’t work as expected. Please stay tuned.
</div>

If you only want to install Syndesis without developing for, there is even an easier without checking out Syndesis into a local Git repository.

You can directly use the standalone installation script [syndesis-install](https://raw.githubusercontent.com/syndesisio/syndesis/master/tools/bin/install-syndesis)
for installing Syndesis. Just download this [script](https://raw.githubusercontent.com/syndesisio/syndesis/master/tools/bin/install-syndesis),
save it as "syndesis-install" and then call it with

``` bash
bash install-syndesis --route $(oc project -q).6a63.fuse-ignite.openshiftapps.com --open
```

Or, if you feel fancy (and trust us), then you can directly install the
latest version of Syndesis by deleting and recreating the current
project with a single line:

``` bash
bash <(curl -sL https://bit.ly/syndesis-install) -p $(oc project -q) -r $(oc project -q).6a63.fuse-ignite.openshiftapps.com -o
```

All you need is to have `bash`, `curl` and `oc` installed and you need
to be connected to an OpenShift cluster.

Use `install-syndesis --help` for a list of options (which is a subset of `syndesis install` described above).
