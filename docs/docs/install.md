# Install Mathislewald backend

The Mathislewald backend is fully containerized. You can easily deploy it on 
other hardware. For that purpose, this repository contains a [Docker compose recipe](https://docs.docker.com/compose/)
that will spin up the entire hardware for you.

## Install Docker

First you need Docker. [Install instructions](https://docs.docker.com/engine/install/) can be found in the 
Docker documentation. 

For Debian, you need to all requirements and add Docker to the package sources, as described in the docs.
Then run

```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Start the backend

Download the repository to a suitable location. Note, that the volumes 
for the several container will be mounted on paths relative to the repository.

```bash
git clone https://github.com/hydrocode-de/mathislewald-backend
cd mathislewald-backend
```

The streamlit application will run with non-root user. Hence, you need to provide details to the user. You can find the information by running:

```bash
id <USER> -u  # get UID
id <USER> -g  # get GID
```

Then set these as `GID` and `UID` environment variables. 


The better approach is to add a `.env` file to the repo. Additionally, you can set the `RESTART=no` to prevent a local deployment from restarting on each host system boot. Alternatively, set `RESTART=always` in production, to let the docker daemon restart the containers on system re-boot and failures: 

```bash
echo "UID=$(id -u)\nGID=$(id -g)\nRESTART=no" > .env
```

Finally you can run the whole infrastructure:

```bash
docker compose up -d
```

To make sure everything is running, you can check the running container:
```bash
docker compose ps
```

which should give something like this:

```
NAME                               IMAGE                                             COMMAND                  SERVICE             CREATED             STATUS              PORTS
mathislewald-backend-geoserver-1   docker.osgeo.org/geoserver:2.22.0                 "/bin/sh -c /opt/sta…"   geoserver           26 minutes ago      Up 26 minutes       0.0.0.0:8080->8080/tcp, :::8080->8080/tcp
mathislewald-backend-upload-1      ghcr.io/hydrocode-de/mathislewald-upload:v0.7.0   "streamlit run app/u…"   upload              19 minutes ago      Up 19 minutes       127.0.0.1:8501->8501/tcp
```