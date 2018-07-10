# AccessMap Orchestration

This repo contains orchestration info for AccessMap's deployments. At the
moment, that entails docker-compose setups for different use cases, primarily
focused on:
  - The production deployment on accessmap.io
  - Testing / research deployments for students / researchers
  - Development environments

## The deployments

### Configuration

Each directory is a self-contained docker-compose setup. Configuration is
controlled by using either a .env file or exporting the necessary environment
variables globally or in the scope of the docker-compose command. An example
.env file is available for each directory as .env.example.

Please note that because all of the services run in a docker container, the
host network is not available without some hacky workarounds. Therefore, if
an environment variable requires a database URI that refers to `localhost`, it
will be looking at the docker container's network, not the host's network.

### Running a deployment

To run any given deployment, just run `docker-compose up` in its directory. To
launch as a background daemon, run `docker-compose -d up`.

The docker-compose files are all version `3`, so make sure you have a recent
version of `docker-compose` (they were initially developed with v1.15).

Running the containers should require only `docker` and `docker-compose`.
Doing some of the follow-up setup may require `curl`.

### Setup

Setting up this deployment is a two-step process, as the analytics
server requires a bit of setup and creating some credentials, and then the
webapp needs to know those credentials and get restarted (i.e. the
docker-compose file needs to be edited).

1. Edit environment variables in the `.env` file. If one doesn't exist, copy
it from `.env_sample`: `cp .env_sample .env`:

  - `RAKAM_CONFIG_LOCK__KEY`: set this to a secret string. This is a 'master'
  password for making new analytics projects, and its endpoint will be public.
  - `MAPBOX_TOKEN`: A Mapbox token for your deployment, lets you use their
  vector tiles for your map.
  - `OPENID_CLIENT_ID`: The client ID value that is registered with OpenToAll accounts.
  This is only necessary if you want logins to work.

If you are running this in production (you probably shouldn't, yet), make sure
to change the database username/password as well using the `RAKAMDB_USER` and
`RAKAMDB_PASSWORD` environment variables. Note that this has no effect if you
have already created and persisted the database (you'll need to change this
info manually in postgres or recreate the database entirely).

2. Add input data for the subproject. This is set up at the 'AMDATA' directory set by
the .env file in the project's directory (this can be copied from .env.sample). The
default directory (if none is set in .env) is `/docker/$PROJECT_data/`, e.g.
`/docker/accessmap_data/`. Note that the data should correspond to the
`config/layers.json` file in the subproject. For example, the `accessmap` project
expects these files to exist:

  - `sidewalks.geojson`

  - `crossings.geojson`

  - `elevator_paths.geojson`

Such data can be generated using the
[accessmap-data](https://github.com/accessmap/accessmap-data) repository.

3. Start the services: `docker-compose up`. Note: to run in production or staging mode,
add docker-compose config cascading, e.g.:
`docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d`

4. Create a new analytics project. Rakam uses web requests for all setup, so
you can use whatever tool you prefer. Also, note that the analytics server is
proxied by the `accessmap-webapp` project, so these commands can be carried
out remotely so long as you have access to it, at `<base_url>/analytics`. The
following example uses `curl` on the local server:

    curl --request POST --url http://localhost:2015/analytics/project/create -d '{"name": "project1", "lock_key": "mylockKey"}'

Note: if you're running the production configuration, use localhost:80 instead.

The response will include a `master_key` and `write_key`. Write these down /
save them in a secure location.

5. Update the .env file and set these variables:

- ANALYTICS_KEY: Set this to the `write_key` value you got in step 3.

6. Stop the services: `docker-compose down` if you started them in daemon mode,
or `ctrl + C` and wait if started them interactively. Rakam/the database
will complain about existing tables, which is fine.

7. Start the services again: `docker-compose up`.

Note that the tiles won't appear until you see the message `pedestrian tiles
built` pop up. It takes ~15-30 seconds for the vector tiles to build.

8. Done! The entire system should now be working.

## Use during development

The easiest way to develop a new service alongside these deployments is to
bind them to a new port, mess with .env, and run in development mode.
