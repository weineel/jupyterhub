# Configuration examples

This section provides examples, including configuration files and tips, for the
following configurations:

- Using GitHub OAuth
- Using nginx reverse proxy

## Using GitHub OAuth

In this example, we show a configuration file for a fairly standard JupyterHub
deployment with the following assumptions:

* Running JupyterHub on a single cloud server
* Using SSL on the standard HTTPS port 443
* Using GitHub OAuth (using oauthenticator) for login
* Users exist locally on the server
* Users' notebooks to be served from `~/assignments` to allow users to browse
  for notebooks within other users' home directories
* You want the landing page for each user to be a `Welcome.ipynb` notebook in
  their assignments directory.
* All runtime files are put into `/srv/jupyterhub` and log files in `/var/log`.

The `jupyterhub_config.py` file would have these settings:

```python
# jupyterhub_config.py file
c = get_config()

import os
pjoin = os.path.join

runtime_dir = os.path.join('/srv/jupyterhub')
ssl_dir = pjoin(runtime_dir, 'ssl')
if not os.path.exists(ssl_dir):
    os.makedirs(ssl_dir)

# Allows multiple single-server per user
c.JupyterHub.allow_named_servers = True

# https on :443
c.JupyterHub.port = 443
c.JupyterHub.ssl_key = pjoin(ssl_dir, 'ssl.key')
c.JupyterHub.ssl_cert = pjoin(ssl_dir, 'ssl.cert')

# put the JupyterHub cookie secret and state db
# in /var/run/jupyterhub
c.JupyterHub.cookie_secret_file = pjoin(runtime_dir, 'cookie_secret')
c.JupyterHub.db_url = pjoin(runtime_dir, 'jupyterhub.sqlite')
# or `--db=/path/to/jupyterhub.sqlite` on the command-line

# use GitHub OAuthenticator for local users
c.JupyterHub.authenticator_class = 'oauthenticator.LocalGitHubOAuthenticator'
c.GitHubOAuthenticator.oauth_callback_url = os.environ['OAUTH_CALLBACK_URL']

# create system users that don't exist yet
c.LocalAuthenticator.create_system_users = True

# specify users and admin
c.Authenticator.whitelist = {'rgbkrk', 'minrk', 'jhamrick'}
c.Authenticator.admin_users = {'jhamrick', 'rgbkrk'}

# start single-user notebook servers in ~/assignments,
# with ~/assignments/Welcome.ipynb as the default landing page
# this config could also be put in
# /etc/jupyter/jupyter_notebook_config.py
c.Spawner.notebook_dir = '~/assignments'
c.Spawner.args = ['--NotebookApp.default_url=/notebooks/Welcome.ipynb']
```

Using the GitHub Authenticator requires a few additional
environment variable to be set prior to launching JupyterHub:

```bash
export GITHUB_CLIENT_ID=github_id
export GITHUB_CLIENT_SECRET=github_secret
export OAUTH_CALLBACK_URL=https://example.com/hub/oauth_callback
export CONFIGPROXY_AUTH_TOKEN=super-secret
# append log output to log file /var/log/jupyterhub.log
jupyterhub -f /etc/jupyterhub/jupyterhub_config.py &>> /var/log/jupyterhub.log
```

## Using a reverse proxy

In the following example, we show configuration files for a JupyterHub server
running locally on port `8000` but accessible from the outside on the standard
SSL port `443`. This could be useful if the JupyterHub server machine is also
hosting other domains or content on `443`. The goal in this example is to
satisfy the following:

* JupyterHub is running on a server, accessed *only* via `HUB.DOMAIN.TLD:443`
* On the same machine, `NO_HUB.DOMAIN.TLD` strictly serves different content,
  also on port `443`
* `nginx` or `apache` is used as the public access point (which means that
  only nginx/apache will bind  to `443`)
* After testing, the server in question should be able to score at least an A on the
  Qualys SSL Labs [SSL Server Test](https://www.ssllabs.com/ssltest/)

Let's start out with needed JupyterHub configuration in `jupyterhub_config.py`:

```python
# Force the proxy to only listen to connections to 127.0.0.1
c.JupyterHub.ip = '127.0.0.1'
```

For high-quality SSL configuration, we also generate Diffie-Helman parameters.
This can take a few minutes:

```bash
openssl dhparam -out /etc/ssl/certs/dhparam.pem 4096
```

### nginx

The **`nginx` server config file** is fairly standard fare except for the two
`location` blocks within the `HUB.DOMAIN.TLD` config file:

```bash
# top-level http config for websocket headers
# If Upgrade is defined, Connection = upgrade
# If Upgrade is empty, Connection = close
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

# HTTP server to redirect all 80 traffic to SSL/HTTPS
server {
    listen 80;
    server_name HUB.DOMAIN.TLD;

    # Tell all requests to port 80 to be 302 redirected to HTTPS
    return 302 https://$host$request_uri;
}

# HTTPS server to handle JupyterHub
server {
    listen 443;
    ssl on;

    server_name HUB.DOMAIN.TLD;

    ssl_certificate /etc/letsencrypt/live/HUB.DOMAIN.TLD/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/HUB.DOMAIN.TLD/privkey.pem;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_dhparam /etc/ssl/certs/dhparam.pem;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_stapling on;
    ssl_stapling_verify on;
    add_header Strict-Transport-Security max-age=15768000;

    # Managing literal requests to the JupyterHub front end
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # websocket headers
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
    }

    # Managing requests to verify letsencrypt host
    location ~ /.well-known {
        allow all;
    }
}
```

If `nginx` is not running on port 443, substitute `$http_host` for `$host` on
the lines setting the `Host` header.

`nginx` will now be the front facing element of JupyterHub on `443` which means
it is also free to bind other servers, like `NO_HUB.DOMAIN.TLD` to the same port
on the same machine and network interface. In fact, one can simply use the same
server blocks as above for `NO_HUB` and simply add line for the root directory
of the site as well as the applicable location call:

```bash
server {
    listen 80;
    server_name NO_HUB.DOMAIN.TLD;

    # Tell all requests to port 80 to be 302 redirected to HTTPS
    return 302 https://$host$request_uri;
}

server {
    listen 443;
    ssl on;

    # INSERT OTHER SSL PARAMETERS HERE AS ABOVE
    # SSL cert may differ

    # Set the appropriate root directory
    root /var/www/html

    # Set URI handling
    location / {
        try_files $uri $uri/ =404;
    }

    # Managing requests to verify letsencrypt host
    location ~ /.well-known {
        allow all;
    }

}
```

Now restart `nginx`, restart the JupyterHub, and enjoy accessing
`https://HUB.DOMAIN.TLD` while serving other content securely on
`https://NO_HUB.DOMAIN.TLD`.


### Apache

As with nginx above, you can use [Apache](https://httpd.apache.org) as the reverse proxy.
First, we will need to enable the apache modules that we are going to need:

```bash
a2enmod ssl rewrite proxy proxy_http proxy_wstunnel
```

Our Apache configuration is equivalent to the nginx configuration above:

- Redirect HTTP to HTTPS
- Good SSL Configuration
- Support for websockets on any proxied URL
- JupyterHub is running locally at http://127.0.0.1:8000

```bash
# redirect HTTP to HTTPS
Listen 80
<VirtualHost HUB.DOMAIN.TLD:80>
  ServerName HUB.DOMAIN.TLD
  Redirect / https://HUB.DOMAIN.TLD/
</VirtualHost>

Listen 443
<VirtualHost HUB.DOMAIN.TLD:443>

  ServerName HUB.DOMAIN.TLD

  # configure SSL
  SSLEngine on
  SSLCertificateFile /etc/letsencrypt/live/HUB.DOMAIN.TLD/fullchain.pem
  SSLCertificateKeyFile /etc/letsencrypt/live/HUB.DOMAIN.TLD/privkey.pem
  SSLProtocol All -SSLv2 -SSLv3
  SSLOpenSSLConfCmd DHParameters /etc/ssl/certs/dhparam.pem
  SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH

  # Use RewriteEngine to handle websocket connection upgrades
  RewriteEngine On
  RewriteCond %{HTTP:Connection} Upgrade [NC]
  RewriteCond %{HTTP:Upgrade} websocket [NC]
  RewriteRule /(.*) ws://127.0.0.1:8000/$1 [P,L]

  <Location "/">
    # preserve Host header to avoid cross-origin problems
    ProxyPreserveHost on
    # proxy to JupyterHub
    ProxyPass         http://127.0.0.1:8000/
    ProxyPassReverse  http://127.0.0.1:8000/
  </Location>
</VirtualHost>
```
