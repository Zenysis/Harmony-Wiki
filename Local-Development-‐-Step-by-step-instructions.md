# Local Development - Step by step instructions

> [!WARNING]
> These instructions go hand in hand with https://github.com/Zenysis/Harmony/tree/sybrand-clean-instructions ; you'll need to be on that branch for everything to work.

## Assumptions

- You're familiar with using terminal commands.
- You're familiar with using git.
- You're familiar with python development.
- You have python version 3.9.18 (or thereabouts) installed.
- You have node v18.17.1, npm 9.8.1 and yarn 1.22.19 (or thereabouts) installed.
- You have Docker 24.0.6 (or thereabouts) installed.
- You're using *nix.

## Clone repository

Clone the [Harmony repository](https://github.com/Zenysis/Harmony):
```
# using SSH
git clone git@github.com:Zenysis/Harmony.git

# change directory to the repository
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
- Create a self_serve folder in the zenysis-harmony-demo bucket; thus you need s3/zenysis-harmony-demo/self_serve to exist.

```
# mb can be used to create a bucket or folder
mc mb s3/zenysis-harmony-demo/self_serve
```
> [!NOTE]
> The pipeline should run without self_serve folder present, but following the steps above removes any confusing error logs that could distract you at this point in the setup.

## Prepare druid

> [!NOTE]  
> You don't have to use a local druid server, but these instructions assume you do. If you're not running druid locally, you'll have to do a few things differently to get indexing to work.

Create a shared &amp; data folder for druid:
```
mkdir -p ~/home/share
mkdir -p ~/data/output
```
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

PYTHONPATH=<source folder, e.g. /Users/jimbo/Harmony>
ZEN_HOME=<source folder, e.g. /Users/jimbo/Harmony>
R77_SRC_ROOT=<source folder, e.g. /Users/jimbo/Harmony>
MC_CONFIG=~/.mc

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

```
python -m venv venv
source ./venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt -r requirements-dev.txt -r requirements-web.txt -r requirements-pipeline.txt
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

You can log in user the default user that is created in a development setup (user: `demo@zenysis.com` password: `zenysis` or create your own user:

```
set -o allexport                                                       
source .env.harmony_demo   
set +o allexport

source ./venv/bin/activate

./scripts/create_user.py --username <EMAIL ADDRESS> --first_name <FIRST NAME> --last_name <LAST NAME> --site_admin --password <PASSWORD>
```


