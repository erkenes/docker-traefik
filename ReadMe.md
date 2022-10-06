# Traefik-Proxy - with additional Cloudflare support

This traefik reverse proxy works with (and without) Cloudflare and has an optional support for traefik hub.

Traefik and other uris that are behind the proxy can be secured with authelia.

## Installation

Run the installation script. It will create the traefik.yml, authelia configuration.yml and docker-compose.yml dynamically.

```shell
./traefik-installer
```

After the installation you need to create a user for authelia.<br />
Run the authelia manager and create a new user that has the group `admins`.

```shell
cd AppData/Authelia
./authelia-manager
```
After that move to the root directory and start the docker containers with:
```shell
docker compose up -d
```


## Use a custom local certificate

Install the RootCA from the directory `certs`. The wildcard domain `*.local.dev` and `local.dev` are now valid.

### Create your own RootCA

Install mkcert

```shell
sudo apt-get update -y
sudo apt-get install wget libnss3-tools

wget https://github.com/FiloSottile/mkcert/releases/download/v1.4.4/mkcert-v1.4.4-linux-amd64

sudo mv mkcert-v1.4.4-linux-amd64 /usr/bin/mkcert
sudo chmod +x /usr/bin/mkcert
```

### Setup local Root CA

```shell
mkcert -install

# Create a local tls certificate
# You could add any domain you need ending by .local.dev
# *.local.dev will create a wildcard certificate so any subdomain in the form like.local.dev will also work.
# Unfortunately you cannot create *.dev wildcard certificate your browser will not allow it.
mkcert -cert-file certs/local.crt -key-file certs/local.key "local.dev" "*.local.dev"
```

---

## Contributing

* [Traefik installer](https://github.com/erkenes/traefik-installer)
* [Authelia Manager](https://github.com/erkenes/authelia-management)