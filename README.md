# Mathislewald backend

This repository contains the Docker compose script to run the infrastructure needed for the Mathislewald App.
Currently this includes two container:

* Geoserver instance, with CORS enabled
* Streamlit application for uploading data

The necessary volumes are created and mounted within the repository root.
**The streamlit application directly loads files into the GeoServer data directory. Thus, you need to protect the application**

The Geoserver will be publicly available on port 8080, while the streamlit only binds to 127.0.0.1. This way, you can use a secured Proxy to handle access.

The streamlit application will run with non-root user. Hence, you need to provide details to the user. You can find the information by running:

```bash
id <USER> -u  # get UID
id <USER> -g  # get GID
```

Then set these as `GID` and `UID` environment variables. 


The better approach is to add a `.env` file to the repo: 

```bash
echo "UID=$(id -u)\nGID=$(id -g)" > .env
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
