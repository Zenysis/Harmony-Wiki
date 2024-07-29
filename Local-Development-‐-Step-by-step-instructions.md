# Local Development - Step by step instructions

## Assumptions

- You're familiar with using terminal commands.
- You're familiar with using git.
- You're familiar with python development.
- You have python version 3.9.18 (or thereabouts) installed.
- You have node v18.17.1, npm 9.8.1 and yarn 1.22.19 (or thereabouts) installed.
- You have Docker 24.0.6 (or thereabouts) installed.
- You're using *nix.

> [!WARNING]
> Python 3.10 and up are NOT supported at this point in time.

### Ports used

_The following ports need to be available for local development:_

* 5000: used by web server (can be problematic on MacOS - as port 5000 is in use by default)
* 5432: used by postgres
* 8080: used by hasura

## Pre-requisites

### Ubuntu 22.04 (Jammy) dev environment

Install pre-requisites

```bash
sudo add-apt-repository ppa:deadsnakes/ppa
```

```bash
sudo apt install python3.9 python3.9-dev python3.9-venv build-essential git cmake curl libtiff-dev sqlite3 wget libsqlite3-dev libcurl4-openssl-dev gcc libpq-dev libnetcdf-dev gfortran libgeos-dev libyaml-dev libffi-dev libbz2-dev apt-transport-https ca-certificates postgresql-client lz4 proj-bin libproj-dev libyaml-dev dtach jq libssl-dev liblz4-tool pigz libopenblas-dev liblapack-dev watchman vim
```

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
```

```bash
nvm install 18
```

```bash
npm install -g yarn
```

### Ubuntu 24.04 (Noble) dev environment

Install pre-requisites

```bash
sudo add-apt-repository ppa:deadsnakes/ppa
```

```bash
sudo apt install python3.9 python3.9-dev python3.9-venv nodejs npm build-essential git cmake curl libtiff-dev sqlite3 wget libsqlite3-dev libcurl4-openssl-dev gcc libpq-dev libnetcdf-dev gfortran libgeos-dev libyaml-dev libffi-dev libbz2-dev apt-transport-https ca-certificates postgresql-client lz4 proj-bin libproj-dev libyaml-dev dtach jq libssl-dev liblz4-tool pigz libopenblas-dev liblapack-dev watchman vim
```

```bash
npm install -g yarn
```

## Clone the [Harmony repository](https://github.com/Zenysis/Harmony):
### Using SSH
```bash
git clone git@github.com:Zenysis/Harmony.git
```
### Using HTTPS
```bash
git clone https://github.com/Zenysis/Harmony.git
```
### Change directory to the repository
```bash
cd Harmony
```

## Prepare S3 compatible server

> [!NOTE]
> You can skip this step if you already have access to a S3 compatible server.

- Create `.env.minio` file
```
MINIO_DATA_FOLDER=<data folder, e.g. /Users/jimbo/minio>
MINIO_ROOT_USER=<specify some user>
MINIO_ROOT_PASSWORD=<specify some password>
```
- Start minio server
```
docker compose --env-file .env.minio -f docker-compose.minio.yaml up --detach
```
- Log into minio server http://localhost:9090/ using the configured root username and password.
- Create a `zenysis-harmony-demo` bucket.
- Create an access key (keep the access key and secret key somewhere safe, we'll be using it in the next step)

## Prepare minio client

- Install the [minio](https://min.io/download) client.
- Create a `local` alias in your minio config.
```
mc alias set local http://localhost:9000 <ACCESS KEY> <SECRET_KEY>
```
- Create a self_serve folder in the zenysis-harmony-demo bucket; thus you need local/zenysis-harmony-demo/self_serve to exist.

```
# mb can be used to create a bucket or folder
mc mb local/zenysis-harmony-demo/self_serve
```
> [!NOTE]
> The pipeline should run without self_serve folder present, but following the steps above removes any confusing error logs that could distract you at this point in the setup.

## Prepare Druid

> [!NOTE]  
> You don't have to use a local Druid server, but these instructions assume you do. If you're not running druid locally, you'll have to do a few things differently to get indexing to work.

> [!CAUTION]
> Running a local Druid server will work fine for smaller datasets, for larger datasets you may run into trouble.

Create a shared &amp; data folder for druid:
```
mkdir -p ~/home/share
mkdir -p ~/data/output
```

### Running druid in a container

> [!NOTE]  
> Running druid containerised on a single server has been done with mixed success.

Create `druid_setup/.env` **replacing variable where appropriate**:
```
SINGLE_SERVER_DOCKER_HOST=
DRUID_SHARED_FOLDER=<druid shared folder, e.g. /Users/jimbo/home/share>
DATA_OUTPUT_FOLDER=<data output folder, e.g. /Users/jimbo/data/output>
```
Start druid:
```
cd druid_setup
make single_server_up
```
You can see if druid is starting up ok by looking at the logs:
```
make single_server_logs
```

You should now be able to visit druid on http://localhost:8888/ (it can take a while to start up)

### Running druid on bare metal

Download the most recent supported version of Druid. Refer to the dist folder in the Zenysis druid extensions to establish the most recently supported version. The `dist` folder in each repository will contain jar files that match the supported version of Druid.

- [druid-arbitrary-granularity - dist](https://github.com/Zenysis/druid-arbitrary-granularity/tree/master/dist)
- [druid-nested-json-parser](https://github.com/Zenysis/druid-nested-json-parser/tree/master/dist)
- [druid-aggregatable-first-last](https://github.com/Zenysis/druid-aggregatable-first-last/tree/master/dist)
- [druid-tuple-sketch-expansion](https://github.com/Zenysis/druid-tuple-sketch-expansion/tree/master/dist)

Follow the instructions on https://druid.apache.org/docs/latest/tutorials/ to get the druid server running.

> [!NOTE]
> You will need Java installed. On Ubuntu: `apt install openjdk-11-jdk`

Install the Zensys druid extensions following the instructs for each extension in turn:
- [druid-tuple-sketch-expansion - installation](https://github.com/Zenysis/druid-tuple-sketch-expansion?tab=readme-ov-file#installation)
- [druid-arbitrary-granularity - installation](https://github.com/Zenysis/druid-arbitrary-granularity?tab=readme-ov-file#installation)
- [druid-aggregatable-first-last - installation](https://github.com/Zenysis/druid-aggregatable-first-last?tab=readme-ov-file#installation)
- [druid-nested-json-parser - installation](https://github.com/Zenysis/druid-nested-json-parser?tab=readme-ov-file#installation)

Upon restarting Druid you should now be able to visit druid on http://localhost:8888/ ; and succesfuly index and query data.

## Prepare an environment file for web and pipeline

Create an environment file `.env.harmony_demo` with the following contents **replacing variables where appropriate**

```
DEFAULT_SECRET_KEY=somesecret

ZEN_ENV=harmony_demo

DRUID_HOST=http://localhost
HASURA_HOST=http://localhost:8088

SQLALCHEMY_DATABASE_URI='postgresql://postgres:zenysis@localhost:5432/harmony_demo-local'

POSTGRES_HOST=localhost

# You can go to https://www.mapbox.com and create an API token.
MAPBOX_ACCESS_TOKEN=<some mapbox access token>

NOREPLY_EMAIL=noreply@<your domain here>
SUPPORT_EMAIL=suppport@<your domain here>

ZEN_HOME=<source folder, e.g. /Users/jimbo/Harmony>
PYTHONPATH=$ZEN_HOME
R77_SRC_ROOT=$ZEN_HOME
# TODO: Is one of these redundant? Why do we have two paths?
MC_CONFIG=~/.mc
MC_CONFIG_PATH=$MC_CONFIG

POSTGRES_PASSWORD=postgres

DRUID_SHARED_FOLDER=<druid shared folder, e.g. /Users/jimbo/home/share>
DATA_OUTPUT_FOLDER=<data output folder, e.g. /Users/jimbo/data/output>

# Assuming you've created a minio alias called "local":
OBJECT_STORAGE_ALIAS=local
```

## Running un-containerised

> [!NOTE]
> From this point on, instructions assume you want to run the web serve and pipeline "natively" (that is, not dockerized) on your local machine.

### Install python dependencies

> [!WARNING]
> Success in this section very much depends on your familiarity with working with Python. Successful installation of the dependencies relies on various pre-requisites being in place, depending on the environment you're using. (e.g. xcode command line tools on mac etc.). Instructions for configuring your particular environment falls outside the scope of this document.

#### Create python virtual environment
```bash
python -m venv venv
```

> [!NOTE]
> On Ubuntu you may have to run `python3.9 -m venv venv` depending on your setup.

#### Activate virtual environment
```bash
source ./venv/bin/activate
```

#### Upgrade python package installer
```bash
python -m pip install --upgrade pip
```

#### Install project dependencies
```bash
python -m pip install wheel==0.43.0
python -m pip install --no-build-isolation -r requirements.txt -r requirements-dev.txt -r requirements-web.txt -r requirements-pipeline.txt
```

### Install node dependencies

```
yarn install
```

### Prepare database

```
set -o allexport                                                       
source .env.harmony_demo   
set +o allexport

source ./venv/bin/activate

yarn init-db harmony_demo --populate_indicators_from_config
```

> [!WARNING]
> There is a known issue, where on first run, a failure may occur. If you re-run the command it should be fine.

### Run pipeline

> [!NOTE]
> As noted above, these instructions assume your running druid on your development machine.

> [!WARNING]
> The index step can be very slow when running druid locally. In addition druid does not seem to run stable on mac silicon, and you may have to restart docker several times / adjust your settings depending on your hardware.

Run process, index and validate (the demo pipeline doesn't have a generate step)
```
set -o allexport                                                       
source .env.harmony_demo   
set +o allexport

source ./venv/bin/activate

./pipeline/harmony_demo/process/process_all

./pipeline/harmony_demo/index/index_all

./pipeline/harmony_demo/validate/validate_all                       
```

> [!NOTE]
> You can check up on druid indexing progress on the druid console at http://localhost:8081/

### Start web server

Start webpack:
```
yarn webpack
```

In a separate terminal, start the flask server:
```
set -o allexport                                                       
source .env.harmony_demo   
set +o allexport

source ./venv/bin/activate

yarn server
```

You should now be able to browse to the login screen on http://localhost:5000/

> [!NOTE]
> If you encounter the following error when visiting the site:
> ```
> {
>   "msg": "Signature verification failed"
> }
> ```
> It's probably due to the `DEFAULT_SECRET_KEY` used by JWT having changed. You can resolve this by deleting cookies, or changing `DEFAULT_SECRET_KEY` to the correct value.

You can log in user the default user that is created in a development setup (user: `demo@zenysis.com` password: `zenysis`) or create your own user:

```
set -o allexport                                                       
source .env.harmony_demo   
set +o allexport

source ./venv/bin/activate

./scripts/create_user.py --username <EMAIL ADDRESS> --first_name <FIRST NAME> --last_name <LAST NAME> --site_admin --password <PASSWORD>
```


