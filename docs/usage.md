# Usage

## At build time

The preferred way to configure `setuptools-scm` is to author
settings in the `tool.setuptools_scm` section of `pyproject.toml`.

It's necessary to use a setuptools version released after 2022.

```toml title="pyproject.toml"
[build-system]
requires = ["setuptools>=64", "setuptools-scm>=8"]
build-backend = "setuptools.build_meta"

[project]
# version = "0.0.1"  # Remove any existing version parameter.
dynamic = ["version"]

[tool.setuptools_scm]
# can be empty if no extra settings are needed, presence enables setuptools-scm
```

That will be sufficient to require `setuptools-scm` for projects
that support PEP 518 ([pip](https://pypi.org/project/pip) and
[pep517](https://pypi.org/project/pep517/)).
Tools that still invoke `setup.py` must ensure build requirements are installed

### Version files

Version files can be created with the ``version_file`` directive.

```toml title="pyproject.toml"
...
[tool.setuptools_scm]
version_file = "pkg/_version.py"
```
Where ``pkg`` is the name of your package.

Unless the small overhead of introspecting the version at runtime via
`importlib.metadata` is a concern or you need a version file in an
alternative format such as plain-text (see ``version_file_template``)
you most likely do _not_ need to write a separate version file; see
the runtime discussion below for more details.

## As cli tool

If you need to confirm which version string is being generated
or debug the configuration, you can install
[setuptools-scm](https://github.com/pypa/setuptools-scm)
directly in your working environment and run:

```commandline
$ python -m setuptools_scm # example from running local after changes
7.1.1.dev149+g5197d0f.d20230727
```

 and to list all tracked by the scm:

```commandline
$ python -m setuptools_scm ls # output trimmed for brevity
./LICENSE
...
./src/setuptools_scm/__init__.py
./src/...
...
```

!!! note "Committed files only"

    currently only committed files are listed, this might change in the future

!!! warning "sdists/archives don't provide file lists"

    Currently there is no builtin mechanism
    to safely transfer the file lists to sdists or obtaining them from archives.
    Coordination for setuptools and hatch is ongoing.

To explore other options, try

```commandline
$ python -m setuptools_scm --help
```

## At runtime

### Python Metadata

The standard method to retrieve the version number at runtime is via
[PEP-0566](https://www.python.org/dev/peps/pep-0566/) metadata using
``importlib.metadata`` from the standard library (added in Python 3.8)
or the
[`importlib_metadata`](https://pypi.org/project/importlib-metadata/)
backport for earlier versions:

```python title="package_name/__init__.py"
from importlib.metadata import version, PackageNotFoundError

try:
    __version__ = version("package-name")
except PackageNotFoundError:
    # package is not installed
    pass
```

### Via your version file

If you have opted to create a Python version file via the standard
template, you can import that file, where you will have a ``version``
string and a ``version_tuple`` tuple with elements corresponding to
the version tags.

```python title="Using package_name/_version.py"
import package_name._version as v

print(v.version)
print(v.version_tuple)
```

### Via setuptools_scm (strongly discouraged)

While the most simple **looking** way to use `setuptools_scm` at
runtime is:

```python
from setuptools_scm import get_version
version = get_version()
```

it is strongly discouraged to call directly into `setuptools_scm` over
the standard Python `importlib.metadata`.

In order to use `setuptools_scm` from code that is one directory deeper
than the project's root, you can use:

```python
from setuptools_scm import get_version
version = get_version(root='..', relative_to=__file__)
```

### Usage from Sphinx


``` {.python file=docs/.entangled/sphinx_conf.py}
from importlib.metadata import version as get_version
release: str = get_version("package-name")
# for example take major/minor
version: str = ".".join(release.split('.')[:2])
```

The underlying reason is that services like *Read the Docs* sometimes change
the working directory for good reasons and using the installed metadata
prevents using needless volatile data there.


### With Docker/Podman


In some situations, Docker may not copy the `.git`  into the container when
building images. Because of this, builds with version inference may fail.

The following snippet exposes the external `.git` directory without copying.
This allows the version to be inferred properly form inside the container
without copying the entire `.git` folder into the container image.

```dockerfile
RUN --mount=source=.git,target=.git,type=bind \
    pip install --no-cache-dir -e .
```
However, this build step introduces a dependency to the state of your local
`.git` folder the build cache and triggers the long-running pip install process on every build.
To optimize build caching, one can use an environment variable to pretend a pseudo
version that is used to cache the results of the pip install process:


```dockerfile
FROM python
COPY pyproject.toml
ARG PSEUDO_VERSION=1 # strongly recommended to update based on git describe
RUN SETUPTOOLS_SCM_PRETEND_VERSION_FOR_MY_PACKAGE=${PSEUDO_VERSION} pip install -e .[test]
RUN --mount=source=.git,target=.git,type=bind pip install -e .
```

Note that running this Dockerfile requires docker with BuildKit enabled
[docs](https://github.com/moby/buildkit/blob/v0.8.3/frontend/dockerfile/docs/syntax.md).

To avoid BuildKit and mounting of the .git folder altogether, one can also pass the desired
version as a build argument.
Note that `SETUPTOOLS_SCM_PRETEND_VERSION_FOR_${NORMALIZED_DIST_NAME}`
is preferred over `SETUPTOOLS_SCM_PRETEND_VERSION`.



## Default versioning scheme

In the standard configuration `setuptools-scm` takes a look at three things:

1. latest tag (with a version number)
2. the distance to this tag (e.g. number of revisions since latest tag)
3. workdir state (e.g. uncommitted changes since latest tag)

and uses roughly the following logic to render the version:


| distance | state     | format                                                               |
|----------|-----------|----------------------------------------------------------------------|
| no       | unchanged | `{tag}`                                                              |
| yes      | unchanged | `{next_version}.dev{distance}+{scm letter}{revision hash}`           |
| no       | changed   | `{tag}+dYYYYMMDD`                                                    |
| yes      | changed   | `{next_version}.dev{distance}+{scm letter}{revision hash}.dYYYYMMDD` |

where `{next_version}` is the next version number after the latest tag

The next version is calculated by adding `1` to the last numeric component of
the tag.

For Git projects, the version relies on  [git describe](https://git-scm.com/docs/git-describe),
so you will see an additional `g` prepended to the `{revision hash}`.


!!! note

    According to [PEP 440](https://peps.python.org/pep-0440/#local-version-identifiers>),
    if a version includes a local component, the package cannot be published to public
    package indexes like PyPI or TestPyPI. The disallowed version segments may
    be seen in auto-publishing workflows or when a configuration mistake is made.

    However, some package indexes such as devpi or other alternatives allow local
    versions. Local version identifiers must comply with [PEP 440](https://peps.python.org/pep-0440/#local-version-identifiers>).

## Semantic Versioning (SemVer)

Due to the default behavior it's necessary to always include a
patch version (the `3` in `1.2.3`), or else the automatic guessing
will increment the wrong part of the SemVer (e.g. tag `2.0` results in
`2.1.devX` instead of `2.0.1.devX`). So please make sure to tag
accordingly.


## Builtin mechanisms for obtaining version numbers

1. the SCM itself (Git/Mercurial)
2. `.hg_archival` files (Mercurial archives)
3. `.git_archival.txt` files (Git archives, see subsection below)
4. `PKG-INFO`


### Git archives

Git archives are supported, but a few changes to your repository are required.

Ensure the content of the following files:

```{ .text file=".git_archival.txt"}

node: $Format:%H$
node-date: $Format:%cI$
describe-name: $Format:%(describe:tags=true,match=*[0-9]*)$
```

Feel free to alter the `match` field in `describe-name` to match your project's
tagging style.

!!! note

    If your git host provider does not properly expand `describe-name`, you may
    need to include `ref-names: $Format:%D$`. But **beware**, this can often
    lead to the git archive's checksum changing after a commit is added
    post-release. See [this issue][git-archive-issue] for more details.


``` {.text file=".gitattributes"}
.git_archival.txt  export-subst
```

Finally, don't forget to commit the two files:
```commandline
$ git add .git_archival.txt .gitattributes && git commit -m "add export config"
```


Note that if you are creating a `_version.py` file, note that it should not
be kept in version control. It's strongly recommended to be put into gitignore.

[git-archive-issue]: https://github.com/pypa/setuptools-scm/issues/806

### File finders hook makes most of `MANIFEST.in` unnecessary

!!! warning "Automatic File Inclusion"

    **`setuptools-scm` automatically provides a setuptools file finder by default.** This means that when you install setuptools-scm, it will automatically include **all SCM-tracked files** in your source distributions (sdist) without requiring a `MANIFEST.in` file.

    This automatic behavior can be surprising if you're not expecting it. The file finder is active as soon as setuptools-scm is installed in your build environment.

`setuptools-scm` implements a [file_finders] entry point
which returns all files tracked by your SCM.
This eliminates the need for a manually constructed `MANIFEST.in` in most cases where this
would be required when not using `setuptools-scm`.

[file_finders]: https://setuptools.pypa.io/en/stable/userguide/extension.html

#### How it works

1. **Automatic Discovery**: When building source distributions (`python -m build --sdist`), setuptools automatically calls the `setuptools-scm` file finder
2. **SCM Integration**: The file finder queries your SCM (Git/Mercurial) for all tracked files
3. **Inclusion**: All tracked files are automatically included in the sdist

#### Controlling file inclusion

**To exclude unwanted files:**

1. **Use `MANIFEST.in`** to exclude specific files/patterns:
   ```
   exclude development.txt
   recursive-exclude tests *.pyc
   ```

2. **Configure Git archive** (for Git repositories):
   ```bash
   # Add to .gitattributes
   tests/ export-ignore
   *.md export-ignore
   ```

3. **Use `.hgignore`** or **Mercurial archive configuration** (for Mercurial repositories)

#### Troubleshooting

**Problem: Unwanted files in my package**
- ✅ **Solution**: Add exclusions to `MANIFEST.in`
- ✅ **Alternative**: Use Git/Mercurial archive configuration

**Problem: Missing files in package**
- ✅ **Check**: Are the files tracked in your SCM?
- ✅ **Solution**: `git add` missing files or override with `MANIFEST.in`

**Problem: File finder not working**
- ✅ **Check**: Is setuptools-scm installed in your build environment?
- ✅ **Check**: Are you in a valid SCM repository?

### Timestamps for Local Development Versions

!!! info "Improved Timestamp Behavior"

    When your working directory has uncommitted changes (dirty), setuptools-scm now uses the **actual modification time of changed files** instead of the current time for local version schemes like `node-and-date`.

    **Before**: Dirty working directories always used current time (`now`)
    **Now**: Uses the latest modification time of changed files, falling back to current time only if no changed files are found

    This provides more stable and meaningful timestamps that reflect when you actually made changes to your code.

**How it works:**

1. **Clean repository**: Uses commit timestamp from SCM
2. **Dirty repository**: Uses latest modification time of changed files
3. **Fallback**: Uses current time if no modification times can be determined

**Benefits:**

- More stable builds during development
- Timestamps reflect actual change times
- Better for reproducible development workflows
