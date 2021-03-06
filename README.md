# Installation

Requirements: docker and docker-compose

To build and deploy actinia, run

```
git clone https://github.com/mundialis/actinia-docker.git
cd actinia-docker
docker-compose -f docker-compose.yml up
```
Now you have a running actinia instance locally! Check with
```
curl http://127.0.0.1:8088/api/v3/version
```

* Having __trouble__? See [How to fix common startup errors](#startup-errors) below.
* Want to __start developing__? Look for [Local dev-setup with docker](#local-dev-setup) below.
* For __production deployment__, see [Production deployment](#production-deployment) below.

On startup, some GRASS GIS locations are created by default but they are still empty. How to get some geodata to start processing, see in [Testing GRASS GIS inside a container](#grass-gis)  below.

# General

This repository is a set of Dockerimages and docker-compose configs.

Ready to use Dockerimages are available at https://hub.docker.com/r/mundialis/actinia-core

__actinia-alpine__
* build and pushed to [`mundialis/actinia`](https://hub.docker.com/repository/docker/mundialis/actinia)
* includes actinia-plugins
* includes GRASS GIS addons
* includes snappy

__actinia-dev__
* useful for local dev setup for actinia and plugins

__actinia-prod__
* example configuration for production deployment (overwrites default config)

__actinia-ubuntu__
* Ubuntu Dockerimage with snappy and some actinia plugins

__actinia-data__
* Shared by all containers, contains common configs and data


<a id="startup-errors"></a>
# How to fix common startup errors
* if a redis db is running locally this will fail. Run and try again:
```
/etc/init.d/redis-server stop
```
* if you see an error like "max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]", run
```
sudo sysctl -w vm.max_map_count=262144
```
  this is only valid on runtime. To change permanently, set vm.max_map_count in /etc/sysctl.conf


<a id="local-dev-setup"></a>
# Local dev-setup with docker

For a dev setup where no actinia plugins are involved, simply follow the [steps explained in actinia-core](https://github.com/mundialis/actinia_core/tree/main/docker#local-dev-setup-with-docker) directly. Instead of from `actinia_core/docker` run the steps from `actinia-docker`.


## Local dev-setup for actinia-core plugins
For a local dev setup for actinia-core and actinia-core plugins
(eg. actinia-actinia-metadata-plugin or actinia-module-plugin), uninstall them
in `actinia-dev/Dockerfile` (see outcommented code) and mount local checkouts in the
container via `docker-compose-dev.yml` file (also see outcommented code).
Build and run like described below and rebuild from your mounted source code:
```
docker-compose -f docker-compose-dev.yml build
docker-compose -f docker-compose-dev.yml run --rm --service-ports --entrypoint sh actinia

(cd /src/actinia_core && python3 setup.py install)
(cd /src/actinia-metadata-plugin && python3 setup.py install)
(cd /src/actinia-module-plugin && python3 setup.py install)

# you can also run tests here, eg.
(cd /src/actinia-module-plugin && python3 setup.py test)

gunicorn -b 0.0.0.0:8088 -w 1 --access-logfile=- -k gthread actinia_core.main:flask_app
```

To avoid cache problems, remove the packaged actinia_core (already done in Dockerfile)
```
pip3 uninstall actinia_core -y
```
If you have other problems with cache, run
```
python3 setup.py clean --all
```

## Local dev-setup for actinia-core plugins with vscode


For a local dev setup for actinia-core and actinia-core plugins with
[vscode](https://code.visualstudio.com/), you need to have the actinia
repositories checked out next to each other, e.g.
```
repos/actinia/
????????? actinia-api
????????? actinia_core
????????? actinia-docker
????????? actinia-metadata-plugin
????????? actinia-module-plugin
```
Then move the `.vscode` folder from this repository one level up and
open the whole actinia folder as workspace in vscode. This can be done by eg. typing `code $PATH_TO_MY_ACTINIA_CHECKOUTS` in a terminal. Then press `F5` and after a few seconds, a browser window should be opened pointing to the version endpoint. For debugging tips, [read the docs](https://code.visualstudio.com/Docs/editor/debugging#_debug-actions).

<a id="grass-gis"></a>
# Testing GRASS GIS inside a container

See this [README](https://github.com/mundialis/actinia_core/tree/main/docker#testing-grass-gis-inside-a-container) for more information.

<a id="production-deployment"></a>
# Production deployment

To run actinia_core in production systems, you can use the docker-compose-prod.yml. Please change before the default redis password in redis_data/config/.redis and inside the actinia.cfg. Also incomment the `build` block in the `docker-compose.yml`.
To start the server, run:

```
docker-compose -f docker-compose.yml build
docker-compose -f docker-compose.yml up -d
```
Then actinia runs at 'http://127.0.0.1:8088' and depending on your server settings might be accessible from outside. Because of this the start.sh is overwritten to not create any user to avoid security vulnerability. You will have to use a clean redis database to avoid stored actinia credentials from previous runs. You have to create the user by yourself by using the built-in actinia-user cli. __Please change below username (-u) and password (-w)__:
```
# list help about the cli tool:
docker-compose -f docker-compose.yml exec actinia \
    actinia-user --help

# create a user and grant permissions to mapsets:
docker-compose -f docker-compose.yml exec actinia \
    actinia-user create -u actinia-core -w actinia-core -r user -g user -c 100000000 -n 1000 -t 6000
docker-compose -f docker-compose.yml exec actinia \
    actinia-user update -u actinia-core -d nc_spm_08/PERMANENT
```
Read more about user roles here: https://actinia.mundialis.de/tutorial/actinia_concepts.html#user-user-roles-and-user-groups

# Testing the actinia server

After deployment, you should be able to access the endpoints.

Examples:

* https://actinia.mundialis.de/latest/version
* https://actinia.mundialis.de/latest/health_check
* requires authorization (actinia user):
    * https://actinia.mundialis.de/api/v3/locations

# Cloud deployment with multiple actinia_core instances

For cloud deployment, you can use the `docker-swarm.sh` script as a starting point.

Shutdown current actinia swarm:
```
docker stack rm actinia_swarm
docker swarm leave --force
```
Start new actinia swarm:
```
bash docker-swarm.sh
```
See the scripts for hints.
