---
title: "conda-pypi internals"
---

# conda pypi

Stay in the conda ecosystem when developing Python packages.

Simplify your CI.

---

# Challenges using conda with pip

## Accidental dependencies
The pip dependency pulls in all of *its* dependencies from pypi, not curated conda.

`[conda environment] -> [package from pip] -> B, C, D`

## Broken environments

pip does not consider conda when installing packages. conda may not be able to upgrade pip packages. Once an environment has pip dependencies it can be hard to go back.

## Hidden build system dependency

pip pulls the build system e.g. setuptools, flit, hatchling, from pypi. It’s tricky to avoid this install path and it’s not in the package’s main dependencies list.

## Extra CI step

conda for most dependencies, but pip install -e for the code under test?

---

# An Emerging Solution

`conda pypi install -e <path>`

Uses Python build hooks to convert Python packages to native conda packages on-the-fly. If a dependency can come from conda, it will come from conda.

---

# How does it work?

---

# From setup.py to pyproject.toml

In the beginning, Python packages were defined with `setup.py`

The installer runs `setup.py`, but code in `setup.py` performs the installation.

In 2013, wheel decoupled `setup.py` from the install. Instead, `setup.py` builds a wheel, and the installer installs the wheel.

In 2015, Thomas Kluyver released `flit`, which built a `pip`-installable wheel without using setuptools; but `pip` didn't understand `flit` source distributions.

In 2017, [PEP 517](https://peps.python.org/pep-0517/) taught `pip` how to ask `flit` or any other build to create a wheel.

In 2021, [PEP 660](https://peps.python.org/pep-0660/) taught `pip` how to ask any build system for an "editable" install, linking a source checkout to a Python environment.

---

# How are Python packages formed?

`pyproject.toml` tells us what we need to build a Python project.
```toml
[build-system]
build-backend = "hatchling.build"
requires = ["hatchling >=1.12.2", "hatch-vcs >=0.2.0",]

[project]
name = "conda"
```
1. Install `build-system.requires`: `"hatchling", "hatch-vcs"`
2. [`hatchling.build`](https://github.com/pypa/hatch/blob/master/backend/src/hatchling/build.py) provides PEP 517 hooks.
3. `hatchling.build.get_requires_for_build_wheel()` gives additional build-system dependencies. Easy to `pip` install these by accident in a conda recipe!
4. `hatchling.build.build_wheel()` creates a wheel.

---

# ... And conda-pypi can build wheels too

```python
def build_pypa(path: Path, output_path, prefix: Path, distribution="editable"):

    builder = ProjectBuilder(path)

    build_system_requires = builder.build_system_requires
    install_missing(build_system_requires) # with conda

    requirements = builder.get_requires_for_build(distribution)
    install_missing(requirements) # with conda

    wheel_path = builder.build(distribution, output_path)
    return wheel_path
```

---

# ... And conda can install them

After building a wheel, `conda-pypi` converts it to a `.conda` package and hands that off to `conda`. Or, `conda` installs the wheel directly - performing the same conversion without an intermediate archive.

For an "editable" package, that ephemeral wheel only contains a reference to your source checkout, which conda installs; but, unlike `pip install -e .`, conda can uninstall or upgrade the package when required.

---

# In CI

```yaml
jobs:
    build:
        steps:
            - name : Install
                run: |
                    conda install -n base conda-pypi
                    conda install <project dependencies,> <hard to get on pypi e.g. postgresql>
                    conda pypi install <only available on pypi>
                    conda pypi install -e ./my-project/
```

---

# In Summary

```bash
% conda install -n base conda-pypi
% conda pypi --help
usage: conda pypi [-n ENVIRONMENT | -p PATH] [--json] [--console CONSOLE] [-v]
                  [-q] [-d] [-y] [-h]
                  COMMAND ...

Install PyPI packages as conda packages

commands:
  The following subcommands are available.

  COMMAND
    install             Install PyPI packages as conda packages
    convert             Build and convert local Python sdists, wheels or
                        projects to conda packages
```

We've focused on `conda-pypi`'s editable mode, but it has more features.

Try it and let us know at https://github.com/conda/conda-pypi