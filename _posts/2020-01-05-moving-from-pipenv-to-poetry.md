---
title: Moving from Pipenv to Poetry
author: Malthe Jørgensen
---

We recently moved from pipenv to poetry. So far we've had a good experience with poetry, 

## Why?

- pipenv was slow (poetry can also be slow too, but it's easier to debug why)
- pipenv had issues for multiple members of our team:
    - It didn't work on WSL (Windows Subsystem for Linux)
    - Another team member had consistent crashes, hangs, and cases where it could just not install the packages

## How?

We used dephell ([https://dephell.readthedocs.io/](https://dephell.readthedocs.io/)) to convert our dependencies from pipenv to Poetry:

```bash
dephell deps convert --from=Pipfile --to-format=poetry --to-path=pyproject.toml
dephell deps convert --from=Pipfile.lock --to-format=poetrylock --to-path=poetry.lock
```

After that you have to add and fill out the attributes `name`, `version`, `description`, and `authors` under `tool.poetry` in `pyproject.toml`:

```
[tool.poetry]
name = "eduflow"
version = "1.0.0"
description = "A flexible, lightweight, and easy-to-use LMS"
authors = ["Peergrade Inc."]
```

Generally installing and updating packages has been much faster for us with poetry, compared to pipenv – if you exclude the pathological cases like "Why is `poetry install` taking forever?" below.

## Poetry 101

- How do I activate the virtual environment?
While it is documented, it wasn't in the quickstart. Luckily, it's the same as in pipenv – just do:
`poetry shell`. I use `Ctrl-D` to get out.
- The equivalent of `pipenv install <package>` is `poetry add <package>`.
The "bare" version `pipenv install` is still `poetry install`
- `poetry update <package>` will not update the package beyond the version stated in `pyproject.toml` . To update a package beyond what's in `pyproject.toml` you can do `poetry install <package>@latest`.
- Why is `poetry install` taking forever?
Stop (`Ctrl-C`), and try doing `poetry install -vvv` instead –– you'll probably see that it's going through **a lot** of versions for a particular package. For us, that package was `awscli`. We had `boto3` set at `1.10.*` whereas `awscli` was just `*`. Since the `boto3` version 1.10 require `s3transfer (>=0.2.0,<0.3.0)` and the recent versions of `awscli` requires `s3transfer (>=0.3.0,<0.4.0)` poetry was probing hundreds of `awscli` versions (starting from the most recent and going backwards) to find one that was compatible with  `s3transfer (>=0.2.0,<0.3.0)` . The fix is to not use `*` for `awscli` but setting a meaningful version range.
This leads me to the next point:
- `*`-requirements are bad
If you're coming from Pipenv you might have a lot of those — especially as dev-dependencies. In my opinion, there's mostly no reason to have those. At least lock it down to a major version like `1.*.*` . Even though having `*` on dev-dependencies won't hurt you in production, it'll still stump your team when `pytest` is silently updated from version 4 to 5, or a different backwards-incompatible package update breaks your usual write-test-debug cycle.
An added bonus – and a big one I would say – is that it'll be much faster for Poetry to resolve dependencies and produce the lockfile.
- Running `poetry shell` gives me `Virtual environment already activated: <path to virtual environment>`.
This is most likely because you've run `deactivate`. Running `deactivate` won't work, instead, you have to quit the shell that was started by `poetry shell`: You can do this by doing Ctrl-D on an empty line.