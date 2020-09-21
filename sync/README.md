# Sync

This directory includes a helper script for synchronizing contents
from specified Tekton repositories to this repository.

To run this script locally, set up a Python 3 environment with appropriate
Google Cloud Platform credentials, and execute the following command:

```bash
pip3 install -r requirements.txt
python3 sync.py
```

## Usage

```bash
       USAGE: sync.py [flags]
flags:

sync.py:
  -c,--config: Config directory
    (default: 'config')

Try --helpfull to get a list of all flags.
```

## Dockerfile

To build the docker file

**Note: If you trying running the container without supplying a config directory it will fail. Only copy specific values instead of the entire directory. We're primarily trying to avoid pulling in config/, since it's confusing thatit will not be used.**

```bash
# You must cd into the correct directory to build the image
docker build -t tekton/web sync/.
```


## Configuration Directory

The config directory should include the configuration for syncing/curating contents from
specific Tekton repositories.

See `pipelines.yaml` and `triggers.yaml` for more instructions. These two
YAML files control the synchronization/curation from the `tektoncd/pipeline`
and `tektoncd/triggers` repositories respectively.

The YAML files here are used by the scripts in `../sync`.


The yaml sync file requires the following schema
```yaml
# Each YAML file under sync/ configures how helper/helper.py synchronizes
# contents of various versions from its source of truth (usually a GitHub
# repository of a Tekton component, such as tektoncd/pipelines) to
# content/ (for the lastest version) and vault/ (for earlier versions).

# The name of the component.
# helper.py will use this value to build directories in content/ and vault/.
component: Foobar
# The order of the component.
displayOrder: 0
# The GitHub repository where documentation resides.
repository: https://github.com/tektoncd/foobar
# The directory in the GitHub repository where contents reside.
docDirectory: docs
# The tags (versions) of contents to sync.
# Note that helper.py and related script reads tags in the order specified in
# the following list; the first entry in tags will automatically become the
# latest version of contents.
tags:
  # The name of the tag in the GitHub repository.
- name: master
  # The name to display on tekton.dev.
  # helper.py will use this value in the version switcher and other places.
  displayName: master
  # Key-value pairs of files to sync, where the key is the original filename
  # and the value is the new filename.
  files:
  - foo.md : bar.md
# To add a new version, append to the list as below
#- name: v0.8.2
#  displayName: v0.8.x
#  files:
#  - myfiles.md: myfiles.md
# The link to the GitHub tag page.
archive: https://github.com/tektoncd/foobar/tags
```

## Mental Model

This is a quick diagram that will help you develop a mental model on how the sync works

![](https://i.imgur.com/UavDy7u.png)

## Running Locally

Steps to run the sync locally:

Step 1
```bash
pip install -r requirements.txt
```
Step 2
```bash
python3 sync/sync.py
```

**Note** This is a [link](../DEVELOPMENT.md) for the steps to run the entire website locally.