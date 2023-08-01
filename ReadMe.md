# Traefik-Proxy - with additional Cloudflare support

Traefik reverse proxy that (optionally) supports for Cloudflare and traefik hub.

## Installation

Update the `.env` file to your purposes and then run

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
