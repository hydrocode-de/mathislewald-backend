# Configure the backend

This guide runs through the backend configuration. Make sure that you 
[install and run](./install.md) the backend docker containers first. 
This guide assumes that both, the GeoServer and streamlit upload application
are running.

## Install a web-server

For this guide, [nginx](https://www.nginx.com/) is used.
The nginx is necessary as a proxy server, to manage access to the streamlit 
application, which can directly upload external, potentially untrusted files.
Further, it's needed to enabled SSL-encryption for the full backend stack.

Refer to the [nginx install instructions](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/#installing-prebuilt-debian-packages) for detailed information. On Debian, run
the following command.

```bash
sudo apt-get install nginx
```

Make sure, that nginx is up and running:

```bash
service nginx status
```

This should show something similar to:

```
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2023-02-10 11:30:08 UTC; 5 days ago
       Docs: man:nginx(8)
   Main PID: 929 (nginx)
      Tasks: 3 (limit: 4575)
     Memory: 20.3M
        CPU: 1.791s
     CGroup: /system.slice/nginx.service
             ├─929 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─930 "nginx: worker process" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""
             └─931 "nginx: worker process" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""

Feb 10 11:30:08 geowwd systemd[1]: Starting A high performance web server and a reverse proxy server...
Feb 10 11:30:08 geowwd systemd[1]: Started A high performance web server and a reverse proxy server.
```

## Configure nginx

### nginx user
Make sure, that nginx is allowed to read the files uploaded by the streamlit application.
For this, you can either add the `www-data` user to the group of the user running the
application, let www-data run the application, or change the user running nginx.

Let's assume that a user `geouser` is running the streamlit application. Then change
`/etc/nginx/nginx.conf` first line from

```
user www-data;
```
to 

```
# user www-data;
user geouser;
```

### Password protection

**The most important step is to secure the streamlit application in production:**

Follow the instruction on [how to create a `.htpasswd` file for nginx](https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-http-basic-authentication/).

Create that file to the location: `/etc/nginx/.htpasswd`, or change the config below,
if you choose another location:

```bash
sudo htpasswd -c /etc/nginx/.htpasswd uploader
```
Then, you will be prompted for the password.

### Configure sites

Next, add a new configuration to the available sites of the webserver:

Create a `/etc/nginx/stes-available/mathis.conf`:

```
server {
	listen 80 default_server;
	listen [::]:80 default_server;

	root /home/geouser/www/html;
	index index.html index.htm index.nginx-debian.html;

    # you need to change this
	server_name geowwd.uni-freiburg.de;

	location / {
		add_header 'Access-Control-Allow-Origin' '*' always;
		add_header 'Access-Control-Allow-Methods' 'GET, OPTIONS' always;
		# First attempt to serve request as file, then
		# as directory, then fall back to displaying a 404.
		try_files $uri $uri/ /index.html;
	}

	location /img {
		add_header 'Access-Control-Allow-Origin' '*' always;
		add_header 'Access-Control-Allow-Methods' 'GET, OPTIONS' always;
		root /home/geouser/www;
	}

    location /geoserver {
                proxy_pass http://127.0.0.1:8080/geoserver;
		proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
        }

	# this seems to be necessary for login behind proxy
	location ^~ /j_spring_security_check {
		proxy_pass http://localhost:8080/geoserver;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;
	}

	location /upload {
		auth_basic 		"Upload Area";
		auth_basic_user_file 	/etc/nginx/.htpasswd;
		client_max_body_size 2000M;
                proxy_pass http://127.0.0.1:8501/upload;
        }

        location /upload/static {
                proxy_pass http://127.0.0.1:8501/upload/static/;
        }

        location /upload/healthz {
                proxy_pass http://127.0.0.1:8501/upload/healthz;
        }

        location /upload/vendor {
                proxy_pass http://127.0.0.1:8501/upload/vendor;
        }

        location /upload/stream {
                proxy_pass http://127.0.0.1:8501/upload/stream;
                proxy_http_version 1.1;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
		proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_read_timeout 86400;
		proxy_buffering off;
	}
}
```

Finally, link this config to the `/etc/nginx/sites-enabled` folder:

```bash
sudo ln -s /etc/nginx/sites-available/mathis.conf /etc/nginx/sites-enabled/mathis.conf
```

### Reload

Last, reload the nginx server and verify that it is still running.

```bash
sudo service nginx reload
service nginx status
```