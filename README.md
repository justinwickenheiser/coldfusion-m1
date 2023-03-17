# coldfusion-m1

A localhost environment setup to run coldfusion on an Apple M1 computer.

## Set up

### Install Docker

[Docker](https://docs.docker.com/get-docker/)

### Clone this repo

```
cd ~/Documents
git clone https://github.com/justinwickenheiser/coldfusion-m1.git coldfusion
cd coldfusion
```

### Create External Docker Volumes

Do this after installing docker and before the first run of docker-sync.
`docker volume create coldfusion`

### Build/Run

Start the coldfusion container
```docker-compose up -d```

### Admin interface
http://localhost:8080/CFIDE/administrator/index.cfm
The password is set in `docker-compose.yml` and setting your own value for `CFCONFIG_ADMINPASSWORD`

Remember to set the onServerStart to use /www/gvsu/Server.cfc, add `bananas` as the mail server, add `/www/gvsu/customtags` as a Custom Tag Path, and add your datasources!

Also, you may need to comment out the entire apache service in the `docker-compose.yml` to view the admin because of silly routing and failing to load resources. Only do this if you are running into weird issues inside the admin.

```yml
...
# apache:
#   image: httpd:latest
#   container_name: apache-for-coldfusion
#   ports:
#     - '80:80'
#   volumes:
#     - ../webroot/gvsu:/www/gvsu/:cached
#     - ./assets/conf:/usr/local/apache2/conf
...
```

## Understanding The Process

There are 2 services that are needed for the container:
- cold (i.e. coldfusion)
  - image: ortussolutions/commandbox
- apache (because the commandbox coldfusion doesn't use apache and therefore the .htaccess files were ignored)
  - image: httpd

### Commandbox

This has 2 parts: [commandbox](https://commandbox.ortusbooks.com/) and [cfconfig](https://cfconfig.ortusbooks.com/).

The `docker-compose.yml` file includes environment variables to tell commandbox how to launch. `APP_DIR` will be your webroot.
You can specify different CF engines. The default is some version of lucee. Set `BOX_SERVER_APP_CFENGINE` to be your desired engine. This project is defaulted to `adobe@2018.0.15+330106`. Checkout commandbox's documentation for more [engine options](https://commandbox.ortusbooks.com/embedded-server/server-versions). Change your `CFCONFIG_ADMINPASSWORD` to be whatever.

```yml
volumes:
  # This maps your local code into the container
  - ../webroot/gvsu:/www/gvsu/:cached
  # this will save all the server admin settings between launches.
  # The left `coldfusion` is the name of the volume that you created in setup
  # and must match the external volume name at the end of the docker-compose.yml
  - coldfusion:/usr/local/lib/serverHome/WEB-INF 
```

### Apache

The `apache` service will act as a load balancer. Notice it is running on port :80 so you would hit localhost/xyz. Your browser client requests will go to apache, and then based on the ProxyReverse settings in the `assets/conf/httpd.conf` will direct the request to the commandbox coldfusion server that is on port :8080.

If you use different values for `container_name` and `ports` in your `docker-compose.yml` file, update the `assets/conf/httpd.conf`. Here are two helpful posts about this:
- [commandbox-with-apache-and-docker](https://community.ortussolutions.com/t/commandbox-with-apache-and-docker/8419/2)
- [How-to-configure-Apache-as-a-reverse-proxy-example](https://www.theserverside.com/blog/Coffee-Talk-Java-News-Stories-and-Opinions/How-to-configure-Apache-as-a-reverse-proxy-example)

```
# assets/conf/httpd.conf

...
# ProxyReverse to pass requests from Client -> Apache -> CommandBox Coldfusion
ProxyPreserveHost On
ProxyPassMatch ^/(.+.cf[cm])(/.*)?$ http://<your_coldfusion_service_container_name>:<your_coldfusion_service_port>/$1$2
ProxyPassReverse / http://<your_coldfusion_service_container_name>:<your_coldfusion_service_port>/
```

If you need to make changes to your apache, update the `assets/conf/httpd.conf` file. This is used as the config in the apache container, since it is listed as a volume. This is needed to override the default settings, such as including mod_rewrite.
```yml
apache:
  ...
  volumes:
    - ../webroot/gvsu:/www/gvsu/:cached
    - ./assets/conf:/usr/local/apache2/conf
```

The `assets/conf/httpd.conf` file in this project has been set up by default for webteam development. The following changes have been made:
- Enabled `mod_rewrite`
- Enabled `proxy_module` and `proxy_http_module` (needed specifically for `ProxyReverse`
- `DirectoryRoot` set to /www/gvsu
- Set `AllowOverride All` for Directory /www/gvsu
- Added `index.cfm` to `DirectoryIndex` (needed to find the /xyz root index.cfm which includes the core framework)
- Added the `ProxyReverse` settings at the very end of the file
