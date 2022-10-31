# used software versions

* Ubuntu server 22.04 LTS
* Nightscout latest released version from 31.10.2022 via standard git clone

# setup Ubuntu server

I installed a fresh Ubuntu Server on a small physical box.

During the text based setup process I 
1. asked to add the OpenSsh server as I used it via another Linux box as frontend.
2. did not select any special components to be added - especially not the docker runtime as this was not wokring as I checked later!

# setup docker on Ubuntu server

## general

I followed the provided documentation on docker.com
1. for setup  https://docs.docker.com/engine/install/ubuntu/ and
2. for linux post installation steps https://docs.docker.com/engine/install/linux-postinstall/

## concrete commands when I was logged in as the standard created user during login (not root)

### setup
``` shell
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo docker run hello-world
```

### post installation
``` shell
sudo usermod -aG docker $USER
```

Logout and login again so that the command can have the desired effect!

The next step is just a test of the installed docker engine and not a setup task.

``` shell
docker run hello-world
```

# setup Nightscout

## general

1. git clone of the latest release of Nightscout from the official repository
2. remove the traeffik.io part as my box is behind the firewall and the automatic certificate retrieval will not work. This is no big hamr as I use nginx reverse proxy anyway.
3. add the port 1337 to be exposed by the docker container

## concrete commands when I was logged in as the standard created user during login (not root)

``` shell
git clone https://github.com/nightscout/cgm-remote-monitor.git
cd cgm-remote-monitor
```

change of docker-compose.yml

adapt TZ to your needs!

If you need access to the MongoDB database from outside you have to add the port 27017 to the list of exposed ports at the end of the file.

``` yml
version: '3'

services:
  mongo:
    image: mongo:4.4
    volumes:
      - ${NS_MONGO_DATA_DIR:-./mongo-data}:/data/db:cached

  nightscout:
    image: nightscout/cgm-remote-monitor:latest
    container_name: nightscout
    restart: always
    depends_on:
      - mongo
    environment:
      ### Variables for the container
      NODE_ENV: production
      TZ: Etc/CET

      ### Overridden variables for Docker Compose setup
      # The `nightscout` service can use HTTP, because we use `traefik` to serve the HTTPS
      # and manage TLS certificates
      INSECURE_USE_HTTP: 'true'

      # For all other settings, please refer to the Environment section of the README
      ### Required variables
      # MONGO_CONNECTION - The connection string for your Mongo database.
      # Something like mongodb://sally:sallypass@ds099999.mongolab.com:99999/nightscout
      # The default connects to the `mongo` included in this docker-compose file.
      # If you change it, you probably also want to comment out the entire `mongo` service block
      # and `depends_on` block above.
      MONGO_CONNECTION: mongodb://mongo:27017/nightscout

      # API_SECRET - A secret passphrase that must be at least 12 characters long.
      API_SECRET: abc0123456789

      ### Features
      # ENABLE - Used to enable optional features, expects a space delimited list, such as: careportal rawbg iob
      # See https://github.com/nightscout/cgm-remote-monitor#plugins for details
      ENABLE: careportal rawbg iob

      # AUTH_DEFAULT_ROLES (readable) - possible values readable, denied, or any valid role name.
      # When readable, anyone can view Nightscout without a token. Setting it to denied will require
      # a token from every visit, using status-only will enable api-secret based login.
      AUTH_DEFAULT_ROLES: denied

      # For all other settings, please refer to the Environment section of the README
      # https://github.com/nightscout/cgm-remote-monitor#environment

    ports:
      - '1337:1337'
```

start the docker container with

``` shell
docker compose up
```

Access the Nightscout installation with a web browser via http://ip-of-your-server:1337.

you can stop the container with
``` shell
docker compose down
```

you can run the container detached from the console that it is not killed during logout with the flag -d at the end of the command.

``` shell
docker compose up -d
```
