There are several Makefiles that can assist in various aspects of debugging, building, deploying and managing the project in general.

For detailed information on each Makefile, refer to the files themselves. The Makefiles are continuously evolving and are the best source of truth for their current state. 

## Build `./Makefile`

The Makefile in the root of the project directory is used primarily for managing docker images.

e.g. building the etl pipeline image:

```bash
make etl_pipeline_build
```

## Deploy `./deploy/Makefile`

The Makefile in the `deploy` folder is used primarily for managing (debugging, deploying, start, stopping etc.) the various docker containers that make up the project.

e.g. starting the web server, and then viewing it's logs:

```bash
cd deploy
make web_up
make web_log
```

## Druid `./druid_setup/Makefile`

The Makefile in the `druid_setup` folder is used for managing a druid, either in cluster form, or on a single server.

e.g. starting druid in a single server setup:
```bash
cd druid_setup
make single_server_up
```