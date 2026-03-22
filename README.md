
# Migrating from uv to pip-tools with pyenv and venv on Linux

This guide walks you through moving from uv to a modern Python development workflow using pyenv, venv, and pip-tools.

---

## Summary

1. Install pyenv and system libraries.
2. Configure `.bashrc` for automatic pyenv load.
3. Install and select Python versions via pyenv.
4. Create project folder and use `pyenv local` for project-specific Python.
5. Create a venv and activate it.
6. Upgrade pip inside venv.
7. Create `requirements.in` with top-level packages.
8. (Optional) Create `pyproject.toml` for metadata and optional pinned dependencies.
9. Compile `requirements.in` → `requirements.txt`.
10. Sync venv with `pip-sync`.
11. Modify dependencies by updating `requirements.in` and re-running compile + sync.
12. Quick Setup from pyproject.toml Using pip-tools
13. Using `pyenv-virtualenv` Instead of `venv`

---

## 1. (Optional) Install pyenv prerequisites

To build Python from source with `pyenv` on Linux, you may install all helpful development libraries. These packages are grouped for clarity:

* **Minimum required**: Needed to compile Python successfully.
* **Helpful / recommended**: Improves functionality or enables common modules.
* **Advanced / optional**: Enables optional modules (GUI, compression, optimizations).

### Minimal installation (just enough to build Python)

```bash
sudo apt update
sudo apt install -y \
    build-essential \
    libssl-dev \
    zlib1g-dev \
    libncurses5-dev
```

> Use the minimal set if you want a lightweight system or CI environment. You can always install additional packages later if some Python modules fail to build.

### Full installation (all helpful and advanced packages)

```bash
sudo apt update
sudo apt install -y \
    build-essential \
    libssl-dev \
    zlib1g-dev \
    libbz2-dev \
    libreadline-dev \
    libsqlite3-dev \
    wget \
    curl \
    llvm \
    libncurses5-dev \
    libncursesw5-dev \
    xz-utils \
    tk-dev \
    libffi-dev \
    liblzma-dev \
    git
```

Package Explanations:
build-essential – Minimum: GCC, g++, make; required to compile Python.\
libssl-dev – Minimum: OpenSSL headers; enables SSL support in Python.\
zlib1g-dev – Minimum: zlib compression support.\
libncurses5-dev – Minimum/Helpful: terminal handling for interactive shells.\


libbz2-dev – Helpful: bzip2 compression support for Python bz2 module.\
libreadline-dev – Helpful: readline support for interactive Python shell.\
libsqlite3-dev – Helpful: SQLite support for Python sqlite3 module.\
wget – Helpful: command-line download tool.\
curl – Helpful: alternative download tool.\
git – Helpful: version control, useful for cloning projects and pyenv.\
libncursesw5-dev – Helpful: wide-character support for terminal handling.\

llvm – Advanced: LLVM tools for optimizations and optional modules.\
libffi-dev – Helpful/Advanced: foreign function interface (ctypes module).\
liblzma-dev – Advanced: LZMA compression support.\
xz-utils – Advanced: xz compression support (Python lzma module).\
tk-dev – Advanced: Tk GUI support (Python tkinter module).\

---

## 2. Install pyenv

You can refer to this github page for advanced installation options:
[pyenv install instructions](https://github.com/pyenv/pyenv-installer)

```bash
curl https://pyenv.run | bash
```

---

## 3. Configure shell to load pyenv automatically

1. Open `.bashrc` in your editor:

```bash
nano ~/.bashrc
```

2. Add the following at the end:

```bash
export PYENV_ROOT="$HOME/.pyenv"
[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

3. Save and exit (`Ctrl+O`, Enter, `Ctrl+X`)
4. Reload shell:

```bash
source ~/.bashrc
```

5. Test installation:

```bash
pyenv --version
pyenv 2.6.26
```

---

## 4. Install and select a Python version

1. List installable versions:

```bash
pyenv install --list
```

2. Install a specific version (example: 3.14.3):

```bash
pyenv install 3.14.3
```

3. Set global default Python:

```bash
pyenv global 3.14.3
```

Check version:
```bash
python --version
Python 3.14.3
```
---

## 5. Create a Python project with a specific Python version

Create project folder
```bash
mkdir <project-name>
cd <project-name>
```

Set the python version
```bash
pyenv local 3.14.3
```

> This creates a `.python-version` file. Any terminal opened in this folder automatically uses Python 3.14.3, overriding the global default.

---

## 6. Create a virtual environment and install pip-tools

Create virtual environment
```bash
python -m venv <name-of-venv>
source <name-of-venv>/bin/activate
```

Upgrade pip:

```bash
python -m pip install --upgrade pip
pip --version
```

Install pip-tools package in the virtual environment.
```bash
pip install pip-tools
```
This ensures that pip-compile and pip-sync commands used in later sections will work immediately inside your venv.

> You can alternatively use pyenv-virtualenv to create the virtualenv with some additional features. This is covered in section 13.
---

## 7. Create `requirements.in` for pip-tools

```bash
nano requirements.in
```

Example top-level packages:

```
fastapi
requests
selenium
```

> Add extras or version constraints as needed, e.g., `requests[security]>=2.32.0`.

---

## 8. (Optional) Create `pyproject.toml` to pin dependencies

```toml
[project]
name = "migrating-this-project"
version = "0.1.0"
description = "Example Python project to demonstrate migration of uv based depencencies, showing all types of dependency constraints"
readme = "README.md"
requires-python = ">=3.14"

dependencies = [
    "fastapi==0.100.0",
    "requests>=2.32.0",
    "uvicorn<=0.25.0",
    "python-dotenv>=1.0.0,<2.0.0",
    "sqlalchemy~=2.0.20",
    "pydantic!=2.1.0",
    "requests[security]>=2.32.0,<3.0.0",
    "sqlalchemy[asyncio]==2.0.20",
    "dataclasses; python_version < '3.7'"
]

[project.urls]
Homepage = ""
Repository = ""

[tool.pip-tools]
notes = """
Managed with pip-tools.
Use requirements.in and pip-compile for reproducible dependencies.
All constraint types are demonstrated here.
"""
```

---

## 9. Compile `requirements.in` to create `requirements.txt`

```bash
pip-compile requirements.in --no-strip-extras
```

**Explanation:**

* `--no-strip-extras` tells **pip-tools** to **keep any extras you specified** in your `requirements.in` when generating `requirements.txt`.

  * For example, if you have `requests[security]>=2.32.0` in `requirements.in`, it will remain as `requests[security]==2.32.5` in the compiled file.
* **Implication:** Extras like `[security]` ensure that optional dependencies (e.g., `pyOpenSSL`, `cryptography`) are installed in your environment. Stripping extras would remove these, potentially breaking functionality that relies on them.
* **Current default behavior:** pip-tools **keeps extras** by default, so using `--no-strip-extras` is optional right now.
* **Future behavior:** In pip-tools version 8.0.0 and later, the default will change to **strip extras**, meaning extras will be removed unless you explicitly pass `--no-strip-extras`.

**Example difference in `requirements.txt`:**

* **With `--no-strip-extras`:**

```text
requests[security]==2.32.5  # Extra dependencies for security are included
urllib3==2.6.3
```

* **With future default `strip-extras`:**

```text
requests==2.32.5  # Extra dependencies are removed
urllib3==2.6.3
```

> Using `--no-strip-extras` ensures that all optional features you specified (like `requests[security]`) remain installed, keeping your environment fully functional.

---

## 10. Sync the virtual environment

```bash
pip-sync
```

**Explanation:**

* `pip-sync` installs all the exact package versions listed in `requirements.txt` into your active virtual environment.
* It also **removes any packages that are installed but not listed** in `requirements.txt`, keeping your environment clean and fully reproducible.

> This ensures your environment matches the pinned dependencies from `pip-compile` exactly.

---

## 11. Modify packages in your environment

1. Add or remove packages in `requirements.in`
2. Re-compile and sync:

```bash
pip-compile requirements.in --no-strip-extras
pip-sync
```

> Always treat `requirements.in` as the source of truth for top-level dependencies.

---

## 12. Quick Setup from pyproject.toml Using pip-tools

If your project already has a `pyproject.toml`, you can quickly set up your environment and install all dependencies deterministically:

1. **Create and activate a virtual environment**:

* Create and activate a venv using the steps in section 6:

2. **Create a `requirements.in` from your `pyproject.toml`** (if it doesn’t exist yet):

* Copy all entries from the `[project] dependencies` section into a new `requirements.in` file.
* Include any extras or environment markers as needed.

3. **Compile pinned dependencies**:

```bash
pip-compile requirements.in --no-strip-extras
```

* This generates `requirements.txt` with exact versions for reproducible installs.

4. **Sync your virtual environment**:

```bash
pip-sync
```

* Installs all pinned packages from `requirements.txt`.
* Removes any extraneous packages not listed.

5. **Verify installation**:

```bash
pip list
pip check
```

> This approach ensures deterministic installs while keeping your project metadata in `pyproject.toml`. For future updates, modify `requirements.in`, re-run `pip-compile`, and then `pip-sync`.

---

## 13. Using `pyenv-virtualenv` Instead of `venv`

You can use **`pyenv-virtualenv`** to manage project-specific Python versions and virtual environments. This can replace the standard `venv` workflow with some added convenience.

### Key Differences

| Feature                        | `venv`                           | `pyenv-virtualenv`                                              |
| ------------------------------ | -------------------------------- | --------------------------------------------------------------- |
| **Creation command**           | `python -m venv <env-name>`      | `pyenv virtualenv <python-version> <env-name>`                  |
| **Python version control**     | Uses currently active Python     | Directly selects Python version for the virtualenv              |
| **Activation**                 | `source <env-name>/bin/activate` | `pyenv activate <env-name>` or auto-activated via `pyenv local` |
| **Directory auto-switching**   | Manual                           | Automatic if `pyenv local <env-name>` is set                    |
| **Integration with pip-tools** | Works normally                   | Works normally after activation                                 |


### Workflow Example

1. **Create a project-specific virtualenv**:

```bash
# Create Python 3.14.3 virtualenv named "myproject-env"
pyenv virtualenv 3.14.3 myproject-env
```

2. **Set the project to use this virtualenv automatically**:

```bash
cd <project-folder>
pyenv local myproject-env
# This creates a .python-version file
# Any new terminal opened in this folder will automatically use this virtualenv
```

3. **Activate the environment (optional if using local pyenv version)**:

```bash
pyenv activate myproject-env
```

4. **Install dependencies with pip-tools** (same as with venv):

```bash
pip install --upgrade pip
pip-compile requirements.in --no-strip-extras
pip-sync
```

### Notes

* `pyenv-virtualenv` eliminates the need to manually create a separate `venv`.
* Auto-switching via `.python-version` makes it easy to manage multiple projects with different Python versions.
* The **pip-tools workflow** (`requirements.in` → `requirements.txt` → `pip-sync`) remains exactly the same.

> Using `pyenv-virtualenv` simplifies Python version management while keeping your dependency workflow fully reproducible.

---
