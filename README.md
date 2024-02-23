# Run a script in a cross-arch Python environment

This action runs a custom script in a cross-arch Python environment using
[QEMU][qemu] and [Docker][docker] for emulation/containerization.

By default an Ubuntu-based environment is used, but you can use a custom
`Dockerfile` to use a different OS.

Here is an example demonstrating how to use it in a workflow with a matrix job:

```yaml
jobs:
  cross-arch-python:
    name: Cross-arch tests with python
    strategy:
      fail-fast: false
      matrix:
        architecture:
          - "arm64"
        ubuntu_version:
          - "20.04"
        python_version:
          - "3.11"

    runs-on: ubuntu-${{ matrix.ubuntu_version }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Test
        uses: frequenz-floss/gh-action-run-python-in-qemu@v0.x.x
        with:
          architecture: ${{ matrix.architecture }}
          script: "scripts/pytest.sh"
          ubuntu_version: ${{ matrix.ubuntu_version }}
          python_version: ${{ matrix.python_version }}
```

And `scripts/test.py` could be something like:

```sh
#!/bin/sh

python -m pytest
```

## Inputs

> [!IMPORTANT]
> At least one of `ubuntu_version` or `dockerfile` inputs must be specified.
>
> When `dockerfile` is not set a `Dockerfile` is provided by the action. To
> test using other OSs you must provide your own `Dockerfile`. You can use the
> [Dockerfile in this action](resources/Dockerfile) (and optionally the
> [entry_point](resources/entrypoint)) as a starting point.

* `architecture`: The architecture to use. Required.

  It must be supported by [QEMU][qemu] and in particular the
  [`docker/setup-qemu-action`](https://github.com/docker/setup-qemu-action)
  action. For example: `arm64`.

* `python_version`: The Python version to use. Required.

  If `dockerfile` is not set, this version should be present as an Ubuntu
  package in provided `ubuntu_version`. For example, `3.11`, `3.12`, etc. The
  package `python{python_version}` will be installed and used.

  If `dockerfile` is set, then the way to specify the Python version is
  up to the `dockerfile`, but it should install the requested Python
  version and set it as the default `python` and `python3` commands.

* `script`: The script to run. Required.

  This is the path to the script to run in the environment. It must be relative
  to the root of the repository and have the executable bit set. For example:
  `scripts/test.sh`.

  Please note that:

  * The script should not depend on where it is located or other files in
    your repository, as it will be copied to the Docker container (inside
    `/usr/local/bin`) and run there.
  
  * Only scripts are supported for now, this should be a file and exist in
    the repository, you cannot run a command directly.

* `pass_env`: The environment variables to pass to the script. Optional.

  The format is `VAR1=VALUE1 VAR2=VALUE2 ...`. For example, `FOO=bar BAZ=qux`.

  The environment variables will be set in the Docker container before running
  the script.

  Due to current limitations, values can't have spaces or other special
  characters interpreted by bash.

* `ubuntu_version`: The Ubuntu version to use. Required unless `dockerfile` is
  set. Default: `""`. For example, `20.04`, `22.04`, etc.

  If `dockerfile` is not specified, a `Dockerfile` will be automatically
  provided using `ubuntu:{ubuntu_version}` as the base [Docker][docker] image
  where `script` will be run.

* `dockerfile`: The Dockerfile to use. Optional unless `ubuntu_version` and
  `python_version` are not set.

  When this is used, the docker context directory will be set to the directory
  containing the `dockerfile`. Default: `""`.

  You `Dockerfile` should accept 3 `ARG`s:

  * `SCRIPT`: It will be the local path of the script, the `Dockerfile` should
    copy it to `/usr/local/bin`, as the action will run it from there.

  * `UBUNTU_VERSION`: The Ubuntu passed by the user, it can be empty, and it
    can be ignored if your Docker image is not based in Ubuntu.

  * `PYTHON_VERSION`: The Python version passed by the user, it can be empty.
    If set the Docker image should provide that Python version when calling
    `python` and `python3`.

## Example using a custom `Dockerfile`

Fetching submodules recursively and using a custom `Dockerfile`:

```yaml
jobs:
  cross-arch-test:
    name: Test in arm64
    runs-on: ubuntu-20.04

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Run test (arm64)
        uses: frequenz-floss/gh-action-run-python-in-qemu@v0.x.x
        with:
          architecture: arm64
          script: docker/test.sh
          dockerfile: docker/Dockerfile.arm64
```

[qemu]: https://www.qemu.org/
[docker]: https://www.docker.com/
