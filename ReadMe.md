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

1. Clone this repository to your server:
   ```shell
   git clone https://github.com/erkenes/docker-traefik.git
   ```

2. Navigate to the repository directory:
   ```shell
   cd traefik-reverse-proxy
   ```

3. Create a `.env` file and configure your settings. You can use the provided `.env.example`
   ```shell
   cp .env.example .env
   ```

4. Change the root domain in the `.env` file to match your domain. Traefik will be available with `trafik.ROOT_DOMAIN_NAME`.
   You also have to change the root domain in the file [AppData/traefik-proxy/traefik.yml](AppData/traefik-proxy/traefik.yml)
   ```yaml
   entryPoints:
     https:
       http:
         tls:
           domains:
             # ToDo: Replace domain
             - main: 'traefik.local.dev'
               sans:
                 - '*.local.dev'
    ```

5. Start Traefik
   ```shell
   docker compose -f docker-compose.yml up -d
   ```

6. Your Traefik reverse proxy is now up and running, ready to route incoming traffic to your web services.

## Optional Cloudflare Integration

You have to follow the upper instructions first.

1. Add your cloudflare api credentials to the secret files
    - `secrets/cf_api_key` for the api key
    - `secrets/cf_email` for your email address

2. Start Traefik
   ```shell
   docker compose -f docker-compose.cloudflare.yml up -d
   ```

3. Your Traefik reverse proxy is now up and running, ready to route incoming traffic to your web services.

## Optional Authentication Server

If you wish to use an authentication server for user authentication, refer to the documentation of [this repository](https://github.com/erkenes/docker-authentik) for setup instructions.

## Usage

o use this Traefik reverse proxy, configure your web services to include the appropriate labels in their Docker Compose files.
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
      - "traefik.http.routers.whoami-rtr.rule=Host(`whoami.$DOMAIN`)"
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

---

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
