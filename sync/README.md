# Sync and Version

This directory includes scripts to manage multiple versions of
documentation from the different Tekton projects.
The `sync` script pulls content from a project, the `versions`
script adds a new version to the `sync` config.

## `sync`

This `sync` script allows synchronizing contents from specified Tekton
repositories to this repository.

To run this script locally, set up a Python 3 environment and execute
the script:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip3 install -r requirements.txt
./sync/sync.py
```

**Note:** Follow [these steps](../DEVELOPMENT.md) to run the entire website locally.

### Usage of Sync

```bash
       USAGE: sync.py [flags]
flags:

sync.py:
  -c,--config: Config directory
    (default: 'config')

Try --helpfull to get a list of all flags.
```

### Configuring Directories

The config directory should include the configuration for syncing/curating contents from
specific Tekton repositories.

See `pipelines.yaml` and `triggers.yaml` for more instructions. These two
YAML files control the synchronization/curation from the `tektoncd/pipeline`
and `tektoncd/triggers` repositories respectively.

The YAML files here are used by the scripts in `../sync`.

The yaml sync file requires the following schema

```yaml
# Each YAML file under sync/ configures how sync/sync.py synchronizes
# contents of various versions from its source of truth (usually a GitHub
# repository of a Tekton component, such as tektoncd/pipelines) to
# content/ (for the lastest version) and vault/ (for earlier versions).

# The name of the component.
# sync.py will use this value to build directories in content/ and vault/. This is used to for the list on the redenred web website.
component: Foobar
# The order of the component.
displayOrder: 0
# The GitHub repository where documentation resides.
repository: https://github.com/tektoncd/foobar
# The directory in the GitHub repository where contents reside.
docDirectory: docs
# The link to the GitHub tag page.
archive: https://github.com/tektoncd/foobar/tags
# The tags (versions) of contents to sync.
# Note that sync.py and related script reads tags in the order specified in
# the following list; the first entry in tags will automatically become the
# latest version of contents.
# To add a new version, append to the list as below
#- name: v0.8.2
#  displayName: v0.8.x
#  files:
#  - myfiles.md: myfiles.md
tags:
  # The name of a tag or branch in the GitHub repository.
- name: master
  # The name to display on tekton.dev.
  # sync.py will use this value in the version switcher and other places.
  displayName: master
  # Dict of folders to sync
  files:
    foo.md : bar.md
```

### Mental Model

This is a quick diagram that will help you develop a mental model on how the sync works.

![logical flow of the sync program](../static/images/mental_model.png)

## Running with Docker

To build the docker file

**Note: If you trying running the container without supplying a config directory it will fail. Only copy specific values instead of the entire directory. We're primarily trying to avoid pulling in config/, since it's confusing thatit will not be used.**

```bash
# You must cd into the correct directory to build the image
docker build -t tekton/web sync/.
```

## `versions`

The `versions` script can be used to add a new version or remove an unwanted
one from the `sync` configurations.
It was designed to be integrated in the release process of Tekton projects.

To run this script locally, set up a Python 3 environment and execute
the script:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip3 install -r requirements.txt
./sync/versions.py add --project <project-name> <version-name>
```

**Note:** Follow [these steps](../DEVELOPMENT.md) to run the entire website locally.

## Usage of Versions

The script provides online help via the `--help` flag.
Use `[command] --help` for help on the specific command (`add` or `rm`).

```bash
$ ./sync/versions.py
Usage: versions.py [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

Commands:
  add  add a new version in the config for the specified project
  rm   remove a version from the config for the specified project
```

## Examples

Adding a new version to the pipeline project:

```bash
./sync/versions.py add v0.18.0 --project pipeline
```

Adding a new minor to the triggers project, and remove the old one:

```bash
./sync/versions.py add v0.9.1 --project triggers
./sync/versions.py rm v0.9.0 --project triggers
```
