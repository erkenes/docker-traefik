# Traefik-Proxy - with additional Cloudflare support

This repository provides a configuration for setting up Traefik as a reverse proxy for websites.
It supports optionally a Cloudflare integration, and can optionally be coupled with a [authentik server](https://github.com/erkenes/docker-authentik) for user authentication.


## Prerequisites

Before you begin, ensure you have the following prerequisites:

- [Docker](https://www.docker.com/) installed and configured on your server.
- [Docker Compose](https://docs.docker.com/compose/install/) installed.
- A registered domain name for your websites.
- (Optional) A [Cloudflare](https://www.cloudflare.com/) account with an API key and email address if you plan to use Cloudflare.


## Getting Started

Copy this repository to your server. The default path is `/docker/traefik`, if you use another path you need to change the absolute path in the `.env` file.
```shell
git clone https://github.com/erkenes/docker-traefik.git /docker/traefik
```

Navigate to the repository directory:
```shell
cd /docker/traefik
```

Copy the `.env` file.
```shell
cp .env.example .env
```


### Create a traefik user to run traefik in rootless mode

Create a new traefik user on the server to run the traefik container as a non-root user.
You can change the user id to any other value, but make sure that the user id is not already in use, and you change the `TRAEFIK_UID` in the `.env` file.
Change the ownership of the traefik directory to the new user and set the permissions of the acme.json file.

```shell
sudo useradd -u 2000 -M -s /usr/sbin/nologin traefik
chown -R 2000:2000 /docker/traefik
chmod 600 /docker-alpine/lib/traefik/acme/*.json
```


### Domain for traefik

Change the domain for traefik in the `.env` file.
```text
TRAEFIK_DASHBOARD_HOST=traefik.YOUR_DOMAIN.com
```


### Path to the absolute directory of the traefik configuration

Change the absolute path to the traefik configuration in the `.env` file.
```text
ROOT_PATH=/docker/traefik
```


### Docker Group-ID

Traefik does not access the docker socket directly, instead it uses a docker-proxy to access the docker socket.
The proxy needs the correct docker group.

Get the GID if the docker group with the following command:
```shell
cat /etc/group | grep docker
```

and set the GID in the `.env` file.
```text
DOCKER_GID=999
```


## CA Certificates

Place all certificates into the folder that is defined in the `.env` file.


# Start the containers

```shell
docker compose up -d
```


### Add a new certificate

To add a new certificate, add the path to the `certs.yaml file`. Traefik loads them automatically.

```yaml
tls:
  certificates:
    # domain1.de
    - certfile: /etc/traefik/certs/domain1.de.crt
      keyfile: /etc/traefik/certs/domain1.de.key
    # domain2.de
    - certfile: /etc/traefik/certs/domain2.de.crt
      keyfile: /etc/traefik/certs/domain2.de.key
```


#### Use a custom local certificate

Install the RootCA from the directory `certs`. The wildcard domain `*.local.dev` and `local.dev` are now valid.


##### Create your own RootCA

Install mkcert

```shell
sudo apt-get update -y
sudo apt-get install wget libnss3-tools

wget https://github.com/FiloSottile/mkcert/releases/download/v1.4.4/mkcert-v1.4.4-linux-amd64

sudo mv mkcert-v1.4.4-linux-amd64 /usr/bin/mkcert
sudo chmod +x /usr/bin/mkcert
```


##### Setup local Root CA

```shell
mkcert -install

# Create a local tls certificate
# You could add any domain you need ending by .local.dev
# *.local.dev will create a wildcard certificate so any subdomain in the form like.local.dev will also work.
# Unfortunately you cannot create *.dev wildcard certificate your browser will not allow it.
mkcert -cert-file certs/local.crt -key-file certs/local.key "local.dev" "*.local.dev"
```


### Optional Cloudflare Integration

You have to follow the upper instructions first.

Add your cloudflare api credentials to the secret files
- `secrets/cf_api_key` for the api key
- `secrets/cf_email` for your email address

**Make sure that there is no empty line at the end of the secret files!**

Set your email address in the [traefik.yml](AppData/traefik-proxy/traefik.yml) file.
```yaml
certificatesResolvers:
  dns-cloudflare:
    acme:
      # ToDo: Change this value with your email address
      email: 'your@mail.com'
```

## Optional Features / Integrations


### Authentication Server

If you wish to use an authentication server for user authentication, refer to the documentation of [this repository](https://github.com/erkenes/docker-authentik) for setup instructions.


## Usage

To use this Traefik reverse proxy, configure your web services to include the appropriate labels in their Docker Compose files.
Consult the Traefik documentation for details on how to configure routing and SSL certificates.

```yaml
version: '3.9'
services:
  whoami:
    image: traefik/whoami
    networks:
      - traefik-proxy
    labels:
      - "traefik.enable=true"
      ## HTTP Routers
      - "traefik.http.routers.whoami-rtr.rule=Host(`whoami.$ROOT_DOMAIN_NAME`)"
      - "traefik.http.routers.whoami-rtr.entrypoints=https"
      - "traefik.http.routers.whoami-rtr.tls=true"
networks:
  traefik-proxy:
    external: true
```


## Troubleshooting

If you encounter issues or need further assistance, please check the logs of the Traefik container for error messages.
Additionally, refer to the documentation for Traefik for detailed configuration options and troubleshooting tips.


## License

This project is licensed under the [MIT License](LICENSE).


## Acknowledgments

- [Traefik](https://traefik.io/): The reverse proxy and load balancer used to manage web traffic.


## Contributing

Contributions are welcome! If you have any improvements, bug fixes, or feature requests, please open an issue or submit a pull request.

---

Happy proxying!
