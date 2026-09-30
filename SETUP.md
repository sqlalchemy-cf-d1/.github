<div align="center">

# Legacy development setup

The original setup guide for the **0.1.0** stack, with all repositories and a local **Apache Superset**.

[Which guide?](#which-guide) · [Requirements](#requirements) · [Setup script](#setup-script) · [Manual setup](#manual-setup) · [Connect to D1](#connect-superset-to-d1) · [Test client](#test-client)

</div>

<br>

> [!WARNING]
> **This guide is legacy.** It is kept for reference and is not the main setup guide. It sets up the **0.1.0** stack from before Superset 6.1.0 shipped its own D1 engine spec. `dbapi-d1`, `superset-engine-d1` and `client` are archived.

## Which guide?

| You want to | Use |
|-------------|-----|
| Connect Superset to D1 | [Quick start](profile/README.md#quick-start) in the org README |
| Work on `sqlalchemy-d1` | [Development](https://github.com/sqlalchemy-cf-d1/sqlalchemy-d1#development) in the sqlalchemy-d1 README |
| Run the 0.1.0 stack with a local Superset | This guide |

## What it sets up

| Repository | Status | Role in the 0.1.0 stack |
|------------|--------|-------------------------|
| [**dbapi-d1**](https://github.com/sqlalchemy-cf-d1/dbapi-d1) | Archived | DBAPI 2.0 driver for D1. |
| [**sqlalchemy-d1**](https://github.com/sqlalchemy-cf-d1/sqlalchemy-d1) | Active | SQLAlchemy 1.4 dialect for D1. Its `main` branch is now 0.2.x, see [step 2](#2-clone-the-repositories). |
| [**superset-engine-d1**](https://github.com/sqlalchemy-cf-d1/superset-engine-d1) | Archived | Superset EngineSpec for D1. Also runs the local Superset. |
| [**client**](https://github.com/sqlalchemy-cf-d1/client) | Archived | Test script for the driver and the dialect. |

## Requirements

* **Python 3.11**, installed with [pyenv](https://github.com/pyenv/pyenv?tab=readme-ov-file#installation). The 0.1.0 packages do not support other versions.
* **Poetry**

```bash
pyenv install 3.11.13
curl -sSL https://install.python-poetry.org | python3 -
exec $SHELL
poetry --version
```

## Setup script

[setup.sh](setup.sh) does steps 2, 3 and 5 of the manual setup **in the current directory**. It clones the repositories, puts `sqlalchemy-d1` on its 0.1.0 release, installs them, and creates and initializes Superset in `superset-engine-d1`.

```bash
mkdir -p ~/dev/d01-project
cd ~/dev/d01-project

# Using curl
bash <(curl -sSL https://raw.githubusercontent.com/sqlalchemy-cf-d1/.github/refs/heads/main/setup.sh)

# Using wget
bash <(wget -qO- https://raw.githubusercontent.com/sqlalchemy-cf-d1/.github/refs/heads/main/setup.sh)
```

Then run `cd superset-engine-d1` and continue at [6. Run Superset](#6-run-superset).

## Manual setup

### 1. Create a project directory

All repositories live side by side in one folder. Change the path for your system.

```bash
mkdir -p ~/dev/d01-project
cd ~/dev/d01-project
```

### 2. Clone the repositories

```bash
git clone https://github.com/sqlalchemy-cf-d1/dbapi-d1.git
git clone https://github.com/sqlalchemy-cf-d1/sqlalchemy-d1.git
git clone https://github.com/sqlalchemy-cf-d1/superset-engine-d1.git
```

The `main` branch of `sqlalchemy-d1` is now 0.2.x. For the 0.1.0 code, check out its release tag:

```bash
git -C sqlalchemy-d1 checkout v0.1.0
```

Directory structure after cloning:

```
d01-project/
 ├── dbapi-d1/
 ├── sqlalchemy-d1/
 └── superset-engine-d1/
```

### 3. Install dependencies

Each repository gets its own virtual environment.

```bash
cd dbapi-d1
poetry install

cd ../sqlalchemy-d1
poetry install

cd ../superset-engine-d1
poetry install
```

### 4. Check where the packages come from

At 0.1.0, `sqlalchemy-d1` and `superset-engine-d1` install the other packages from **PyPI**, not from the local clones. Changes in `dbapi-d1` or `sqlalchemy-d1` do not reach Superset until they are published. Only `client` uses the local clones.

From `superset-engine-d1`:

```bash
poetry show sqlalchemy-d1
```

It should show version `0.1.0`.

### 5. Set up Superset

Superset runs from the `superset-engine-d1` repository. Create two files in its root:

```env
# superset-engine-d1/.env
FLASK_APP=superset.app:create_app()
```

```python
# superset-engine-d1/superset_config.py
SECRET_KEY = "<generate secure key>"
SQLALCHEMY_DATABASE_URI = "sqlite:////tmp/superset.db"
```

Generate the key with `openssl rand -hex 16`. Then initialize Superset:

```bash
export SUPERSET_CONFIG_PATH="$(pwd)/superset_config.py"
poetry run superset db upgrade
poetry run superset fab create-admin \
    --username admin \
    --firstname Superset \
    --lastname Admin \
    --email admin@example.com \
    --password admin
poetry run superset init
```

### 6. Run Superset

```bash
export SUPERSET_CONFIG_PATH="$(pwd)/superset_config.py"
poetry run superset run -p 8088 --with-threads --reload --debugger
```

Superset is now at `http://localhost:8088`.

> [!IMPORTANT]
> Run the `export SUPERSET_CONFIG_PATH=...` line again in every new shell, or the server will not work.

## Connect Superset to D1

You need:

* Your Cloudflare **account ID**
* A Cloudflare [**API token**](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/) with D1 read permission
* Your D1 **database ID**
* Some sample data in the database. Run `CREATE TABLE` and `INSERT` statements in the database's console on the Cloudflare D1 dashboard.

In the local Superset:

1. Log in. The user and password are both `admin` if you followed the steps above.
2. Open **Settings > Database Connections > + Database**.
3. In **Choose a database**, pick **Other**.
4. Fill in the SQLAlchemy URI as `d1://<CF_ACCOUNT_ID>:<CF_API_TOKEN>@<D1_DB_ID>`.
5. Click **Test Connection**. When it works, click **Connect**.
6. Open **+ > Data > Create Dataset**.
7. Pick **D1** as the database, **main** as the schema, and your table.
8. Pick a chart that fits your data. It should show a preview.

If this works, the setup is done. If anything breaks, check the server logs.

## Test client

The archived [client](https://github.com/sqlalchemy-cf-d1/client) repository has a `main` script that checks the driver and the dialect against a real D1 database. It has been replaced by the unit and integration tests in `sqlalchemy-d1`.

`client` installs `dbapi-d1` and `sqlalchemy-d1` from the local clones, so `sqlalchemy-d1` must be on the 0.1.0 release from [step 2](#2-clone-the-repositories). First run the SQL in the comment at the top of `src/client/main.py` in your D1 database's console. Then clone `client` next to the other repositories:

```bash
cd ~/dev/d01-project
git clone https://github.com/sqlalchemy-cf-d1/client.git
cd client
poetry install
```

Copy the sample credentials file and fill in `CF_ACCOUNT_ID`, `CF_API_TOKEN` and `D1_DB_ID`:

```bash
cp .env.sample .env
```

Run it:

```bash
poetry run client
```

> [!WARNING]
> Run the script **through Poetry**. `python src/client/main.py` on its own does not find the packages.

## Tests

Each repository has its own unit tests:

```bash
poetry run pytest
```

> [!NOTE]
> Make sure editors and IDEs use the Poetry virtual environment of each repository for linting and running code.
