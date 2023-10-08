# Fix flakeheaven error

## Understand the problem

flakeheaven stopped working on 2023-10-08.

```console
$ pre-commit run flakeheaven --all-files
flakeheaven..............................................................Failed
- hook id: flakeheaven
- exit code: 1

Traceback (most recent call last):
  File "/home/isme/.cache/pypoetry/virtualenvs/botocove-r3vy5Ybn-py3.8/bin/flakeheaven", line 5, in <module>
    from flakeheaven import entrypoint
  File "/home/isme/.cache/pypoetry/virtualenvs/botocove-r3vy5Ybn-py3.8/lib/python3.8/site-packages/flakeheaven/__init__.py", line 5, in <module>
    from ._cli import entrypoint, flake8_entrypoint
  File "/home/isme/.cache/pypoetry/virtualenvs/botocove-r3vy5Ybn-py3.8/lib/python3.8/site-packages/flakeheaven/_cli.py", line 9, in <module>
    from .commands import COMMANDS
  File "/home/isme/.cache/pypoetry/virtualenvs/botocove-r3vy5Ybn-py3.8/lib/python3.8/site-packages/flakeheaven/commands/__init__.py", line 5, in <module>
    from ._baseline import baseline_command
  File "/home/isme/.cache/pypoetry/virtualenvs/botocove-r3vy5Ybn-py3.8/lib/python3.8/site-packages/flakeheaven/commands/_baseline.py", line 6, in <module>
    from .._patched import FlakeHeavenApplication
  File "/home/isme/.cache/pypoetry/virtualenvs/botocove-r3vy5Ybn-py3.8/lib/python3.8/site-packages/flakeheaven/_patched/__init__.py", line 2, in <module>
    from ._app import FlakeHeavenApplication
  File "/home/isme/.cache/pypoetry/virtualenvs/botocove-r3vy5Ybn-py3.8/lib/python3.8/site-packages/flakeheaven/_patched/_app.py", line 10, in <module>
    from flake8.options.config import ConfigParser, get_local_plugins
ImportError: cannot import name 'ConfigParser' from 'flake8.options.config' (/home/isme/.cache/pypoetry/virtualenvs/botocove-r3vy5Ybn-py3.8/lib/python3.8/site-packages/flake8/options/config.py)
```

It happened after I reinstalled the virtualenv.

Poetry selected prehistoric flakeheaven version 0.11.0. The [changelog](https://github.com/flakeheaven/flakeheaven/blob/4591fd3dd04c8180d702e5eab3ec25947d7d3393/CHANGELOG.md) stops at 0.11.1!

```console
$ poetry install
Creating virtualenv botocove-r3vy5Ybn-py3.8 in /home/isme/.cache/pypoetry/virtualenvs
Updating dependencies
Resolving dependencies... (11.0s)

Package operations: 76 installs, 0 updates, 0 removals

  ...
  • Installing flakeheaven (0.11.0)
  ...

Writing lock file

Installing the current project: botocove (1.7.3)
```

The latest CI run ([#73](https://github.com/iainelder/botocove/actions/runs/6443200026/job/17494862297#logs)) in my project fork started at 2023-10-07T20:06Z. That run installed flakeheaven 3.3.0 and the pre-commit check passed.

> Install dev packages
>
> ```text
>   • Installing flakeheaven (3.3.0)
> ```
>
> Lint with flakeheaven
>
> ```text
> flakeheaven..............................................................Passed
> ```

I'm unsure why Poetry now has selected an old version, but it's probably because `pyproject.toml` doesn't constrain any dev version dependencies.

Botocove is a library and so is polite to not constrain the dependencies of an application that uses botocove without a good reason.

The dev dependencies don't affect the library users, and do affect the botocove maintainers. As a maintainer I don't want Poetry to make weird decisions like this. I want it to choose the latest version of every dev tool unless there is a good reason not to. If there is a good reason, I'll declare it in the dependencies with a comment.

And it seems there is no good reason to choose the ancient version of flakeheaven because I am able to upgrade flakeheaven to the same version 3.3.0 used by the CI. As a side effect it downgrades a some transitive dependencies that I don't care about.

```console
$ poetry update flakeheaven
Updating dependencies
Resolving dependencies... (3.3s)

Package operations: 0 installs, 7 updates, 0 removals

  • Downgrading mccabe (0.7.0 -> 0.6.1)
  • Downgrading pycodestyle (2.9.1 -> 2.8.0)
  • Downgrading pyflakes (2.5.0 -> 2.4.0)
  • Downgrading flake8 (5.0.4 -> 4.0.1)
  • Downgrading flake8-eradicate (1.5.0 -> 1.4.0)
  • Updating flakeheaven (0.11.0 -> 3.3.0)
  • Downgrading pep8-naming (0.13.3 -> 0.13.2)

Writing lock file
```

The flakeheaven pre-commit check passes using version 3.3.0. All the other checks pass too.

```console
$ pre-commit run --all-files
trim trailing whitespace.................................................Passed
check python ast.........................................................Passed
check for case conflicts.................................................Passed
debug statements (python)................................................Passed
check yaml...............................................................Passed
markdownlint-cli2........................................................Passed
isort (python)...........................................................Passed
black....................................................................Passed
type annotations not comments............................................Passed
check for eval().........................................................Passed
use logger.warning(......................................................Passed
pytest...................................................................Passed
flakeheaven..............................................................Passed
mypy.....................................................................Passed
```

## Constrain dev dependencies

Set the minimum version for each dev dependencies to be the current latest version. This is Poetry's default behavior when adding a new dependency.

Commit `8bbe6217` removed the version constraints from all dependencies in `pyproject.toml`. It's the right thing to do for the application dependencies, but it's counter-productive for the dev dependencies.

Today (2023-10-08) the latest versions of the dev dependencies are:

* [pytest](https://github.com/pytest-dev/pytest/releases): 7.4.2
* [pytest-mock](https://github.com/pytest-dev/pytest-mock/releases): 3.11.1
* [isort](https://github.com/PyCQA/isort/releases): 5.12.0
* [black](https://github.com/psf/black/releases): 23.9.1
* [flake8-bugbear](https://github.com/PyCQA/flake8-bugbear/releases): 23.9.16
* [flake8-builtins](https://github.com/gforcada/flake8-builtins/tags): 2.1.0
* [flake8-comprehensions](https://github.com/adamchainz/flake8-comprehensions/tags): 3.14.0
* [flake8-eradicate](https://github.com/wemake-services/flake8-eradicate/releases): 1.4.0
* [flake8-isort](https://github.com/gforcada/flake8-isort/tags): 6.1.0
* [flake8-mutable](https://github.com/ebeweber/flake8-mutable/tags): 1.2.0
* [flake8-pytest-style](https://github.com/m-burst/flake8-pytest-style/tags): 1.7.2
* [pep8-naming](https://github.com/PyCQA/pep8-naming/releases): 0.13.3
* [flake8-print](https://github.com/jbkahn/flake8-print): 5.0.0
* [mypy](https://github.com/python/mypy/tags): 1.5.1
* [pre-commit](https://github.com/pre-commit/pre-commit/releases): 3.4.0
* [flakeheaven](https://github.com/flakeheaven/flakeheaven/releases): 3.3.0
* [moto](https://github.com/getmoto/moto/releases): 4.2.5
* [pytest-randomly](https://github.com/pytest-dev/pytest-randomly/tags): 3.15.0

I set these versions in `pyproject.toml`. Then to prepare a clean installation, I delete the virtualenv listed by `poetry env info` and delete `poetry.lock` in the working directory.

Now `poetry install` fails with a version conflict.

```console
$ poetry install
Creating virtualenv botocove-HePr59o--py3.8 in /home/isme/.cache/pypoetry/virtualenvs
Updating dependencies
Resolving dependencies... (5.7s)

Because no versions of flakeheaven match >3.3.0,<4.0.0
 and flakeheaven (3.3.0) depends on flake8 (>=4.0.1,<5.0.0), flakeheaven (>=3.3.0,<4.0.0) requires flake8 (>=4.0.1,<5.0.0).
And because pep8-naming (0.13.3) depends on flake8 (>=5.0.0)
 and no versions of pep8-naming match >0.13.3,<0.14.0, flakeheaven (>=3.3.0,<4.0.0) is incompatible with pep8-naming (>=0.13.3,<0.14.0).
So, because botocove depends on both pep8-naming (^0.13.3) and flakeheaven (^3.3.0), version solving failed.
```

## Resolve conflict over flake8 between flakeheaven and pep8-naming

[pep8-naming version 0.13.3 release notes](https://github.com/PyCQA/pep8-naming/releases) say "formally require flake8 5.0.0 or later."

[Issue 132](https://github.com/flakeheaven/flakeheaven/issues/132) in the flakeheaven project requests support for flake8 5.

I post a comment there to describe the conflict.

---

[botocove](https://github.com/connelldave/botocove/blob/master/pyproject.toml) depends on flakeheaven and pep8-naming.

I want botocove to use the latest version of its development dependencies.

[flakeheaven 3.3.0](https://github.com/flakeheaven/flakeheaven/releases) needs flake8 4.

[pep8-naming 0.13.3](https://github.com/PyCQA/pep8-naming/releases) needs flake8 5.

So I can't install these latest versions together because they conflict over flake8.

To solve this specific conflict I'm going to ask the pep8-naming maintainer whether it can also support flake8 4. Otherwise, I may have to downgrade pep8-naming or find some other solution.

If flake8 5 is the future then flakeheaven needs to adapt to it to remain relevant.

Use this `pyproject.toml` to reproduce the conflict with the [Poetry dependency manager](https://python-poetry.org/).

```toml
[tool.poetry]
name = "test"
version = "0"
description = ""
authors = [""]

[tool.poetry.dependencies]
python = "^3.8"

[tool.poetry.dev-dependencies]
pep8-naming = ">=0.13.3"
flakeheaven = ">=3.3.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

```console
$ poetry install
Updating dependencies
Resolving dependencies... (0.3s)

Because no versions of flakeheaven match >3.3.0
 and flakeheaven (3.3.0) depends on flake8 (>=4.0.1,<5.0.0), flakeheaven (>=3.3.0) requires flake8 (>=4.0.1,<5.0.0).
And because pep8-naming (0.13.3) depends on flake8 (>=5.0.0)
 and no versions of pep8-naming match >0.13.3, flakeheaven (>=3.3.0) is incompatible with pep8-naming (>=0.13.3).
So, because test depends on both pep8-naming (>=0.13.3) and flakeheaven (>=3.3.0), version solving failed.
```

---

I use [GitHub's blame](https://github.com/PyCQA/pep8-naming/blame/ab9ffbe68a387c21c793ff6071ea6d3dc5c5d669/setup.py#L45) on pep8-naming's dependency on flake8 to discover that the maintainers dropped support for flake8 4 in [PR 210](https://github.com/PyCQA/pep8-naming/pull/210).

I won't bother asking them to restore support for flake8 4.

flake8 released the last version of the 4 series on 2021-10-11. The latest general version is 6.0.2, released 2023-07-29.

It doesn't look like there is much progress in the flakeheaven project to stay up to date with flake8. The long-term solution then is to find a modern replacement for flakeheaven.

The short-term solution is to use the latest version of pep8-naming that supports flake8 4.

---

I got quick feedback from @pwoolvett, the flakeheaven maintainer: use [ruff](https://github.com/astral-sh/ruff).

> I'm focusing my efforts on ruff :), and don't have time anymore to make the required upgrades here, specially as the refactor for flake V5 is non trivial.
>
> I suggest ruff, although depending on the complexity of your lint it might take you some time. Alternatively, as you mentioned pinnning down pep8-naming should work.
>
> If you're willing to take a shot on the migration though, you're more than welcome!

And from @berzi, who already migrated:

> In my experience the move to ruff is rather painless, it can even read the codes you have configured for flake8 & friends. The only issue is if you have custom linting rules, because ruff doesn't support those yet. Good luck with your project. :)

ruff is another Python linter. It seems to be a replacement not just for flakeheaven but for flake8.

---

When I used Poetry to upgrade flakeheaven from 0.11.1 to 3.3.0, it downgraded pep8-naming to 0.13.2. That pep8-naming version must be the last version to formally support flake8 4.

I specify pep8-naming 0.13.2 exactly in `pyproject.toml`.

## Resolve flake8-bugbear conflict with Python 3.8.0

I appear to have solved the conflict over flake8. Now I discover that flake8-bugbear conflicts with botocove's minimum Python version (3.8.0).

```console
$ poetry install
Creating virtualenv botocove-HePr59o--py3.8 in /home/isme/.cache/pypoetry/virtualenvs
Updating dependencies
Resolving dependencies... (5.8s)

The current project's Python requirement (>=3.8,<4.0) is not compatible with some of the required packages Python requirement:
  - flake8-bugbear requires Python >=3.8.1, so it will not be satisfied for Python >=3.8,<3.8.1

Because flake8-bugbear (23.9.16) requires Python >=3.8.1
 and no versions of flake8-bugbear match >23.9.16, flake8-bugbear is forbidden.
So, because botocove depends on flake8-bugbear (>=23.9.16), version solving failed.

  • Check your dependencies Python requirement: The Python requirement can be specified via the `python` or `markers` properties

    For flake8-bugbear, a possible solution would be to set the `python` property to ">=3.8.1,<4.0"

    https://python-poetry.org/docs/dependency-specification/#python-restricted-dependencies,
    https://python-poetry.org/docs/dependency-specification/#using-environment-markers
```

Use [GitHub blame](https://github.com/PyCQA/flake8-bugbear/blame/f8fe3c4b3681dd3e5e4cc18daa84fdfb3b1892c2/pyproject.toml#L38) on flake8-bugbear's `requires-python` setting in `pyproject.toml`.

flake8-bugbear started to require Python 3.8.1 in [PR 368](https://github.com/PyCQA/flake8-bugbear/pull/368). It does this "to match flake8."

Use [GitHub blame](https://github.com/PyCQA/flake8/blame/faef3587480ab621dd2fd2d87ec36fc479749a90/setup.cfg#L34) on flake8's `python_requires` setting in `setup.cfg`.

flake8 started to require Python 3.8.1 in [PR 1741](https://github.com/PyCQA/flake8/pull/1741). @asottile says "3.8.0 is ancient, and excluded due to broken importlib.metadata as well."

I'm not sure what he means by the broken `importlib.metadata`.

Search [Python 3.8 changelog](https://docs.python.org/3.8/whatsnew/changelog.html) for all mentions of "importlib" in 3.8.0 and 3.8.1. It doesn't explain why flake8 needs 3.8.1.

> Python 3.8.1 final
>
> * bpo-39022: Update importlib.metadata to include improvements from importlib_metadata 1.3 including better serialization of EntryPoints and improved documentation for custom finders.
>
> Python 3.8.0 release candidate 1
>
> * bpo-38121: Update parameter names on functions in importlib.metadata matching the changes in the 0.22 release of importlib_metadata.
> * bpo-38086: Update importlib.metadata with changes from importlib_metadata 0.21
> * bpo-38010: In importlib.metadata sync with importlib_metadata 0.20, clarifying behavior of files() and fixing issue where only one requirement was returned for requires() on dist-info packages.
>
> Python 3.8.0 beta 3

> * bpo-37697: Syncronize importlib.metadata with importlib_metadata 0.19, improving handling of EGG-INFO files and fixing a crash when entry point names contained colons.
> * bpo-37521: Fix importlib examples to insert any newly created modules via importlib.util.module_from_spec() immediately into sys.modules instead of after calling loader.exec_module().
>
> Python 3.8.0 beta 1
>
> * bpo-34632: Introduce the importlib.metadata module with (provisional) support for reading metadata from third-party packages.

Anyway, if I want to use the latest version of flake8 I need to drop support for Python 3.8.0.

I change the Python version constraint.

```toml
# flake8 needs >=3.8.1. flakeheaven needs <4.0.
# I don't like the dev dependencies to restrict the application Python versions,
# but I don't know how to avoid it.
python = ">=3.8.1,<4.0"
```

## Resolve conflict over flake8 version between flakeheaven and flake8-bugbear

I'm deep in dependency hell right now.

flake8-bugbear 23.9.16 requires flake8 6.

```console
$ poetry install
Creating virtualenv botocove-HePr59o--py3.8 in /home/isme/.cache/pypoetry/virtualenvs
Updating dependencies
Resolving dependencies... (1.9s)

Because no versions of flakeheaven match >3.3.0
 and flakeheaven (3.3.0) depends on flake8 (>=4.0.1,<5.0.0), flakeheaven (>=3.3.0) requires flake8 (>=4.0.1,<5.0.0).
And because flake8-bugbear (23.9.16) depends on flake8 (>=6.0.0)
 and no versions of flake8-bugbear match >23.9.16, flakeheaven (>=3.3.0) is incompatible with flake8-bugbear (>=23.9.16).
So, because botocove depends on both flake8-bugbear (>=23.9.16) and flakeheaven (>=3.3.0), version solving failed.
```

[Version 23.3.23 of flake8-bugbear](https://github.com/PyCQA/flake8-bugbear/releases/tag/23.3.23) introduced the need for at least flake8 6.

The previous [version 23.3.12](https://github.com/PyCQA/flake8-bugbear/releases/tag/23.3.12) needs [at least flake8 3](https://github.com/PyCQA/flake8-bugbear/blob/63acfdc6f75d1f6bb4b16e7bed24bdc6ab7b886e/pyproject.toml#L39).

flake8-bugbear 23.3.12 should be compatible with flakeheaven 3.3.0. This also removes the need to change the Python version constraint.

With this constraint Poetry installs all the dependencies.

## Prepare PR

See [PR 78](https://github.com/connelldave/botocove/pull/78).

## Share workaround

Until Dave Connell merges the PR I can use this command on another branch to install a working version of flakeheaven.

```bash
bash -c '
    set -e
    rm -rf poetry.lock "$(poetry env info --path)"
    poetry install
    poetry update flakeheaven
    pre-commit run flakeheaven --all-files
'
```

The last two commands give this output:

```text
Updating dependencies
Resolving dependencies... (2.4s)

Package operations: 0 installs, 7 updates, 0 removals

  • Downgrading mccabe (0.7.0 -> 0.6.1)
  • Downgrading pycodestyle (2.9.1 -> 2.8.0)
  • Downgrading pyflakes (2.5.0 -> 2.4.0)
  • Downgrading flake8 (5.0.4 -> 4.0.1)
  • Downgrading flake8-eradicate (1.5.0 -> 1.4.0)
  • Updating flakeheaven (0.11.0 -> 3.3.0)
  • Downgrading pep8-naming (0.13.3 -> 0.13.2)

Writing lock file
flakeheaven..............................................................Passed
```
