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

Copy this repository to your server. The default path is `/docker/traefik`.
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

Edit the `.env` file to configure the Traefik proxy. The file contains the following variables:

- `CERT_PATH`: Path where your own certificates are stored. (e.g. if you buy a certificate from a CA)
- `TRAEFIK_STORAGE_PATH`: Path where traefik should store its data (e.g. access and traefik logs).
- `ACME_STORAGE_PATH`: Path where the client configuration of the certificate resolver is stored.
- `HTPASSWD_PATH`: Path where the htpasswd files are stored.
- `ROOT_DOMAIN_NAME`: The root domain name for the Traefik proxy.
- `TRAEFIK_DASHBOARD_HOST`: The domain name for the Traefik dashboard.
- `TRAEFIK_DASHBOARD_CERT_RESOLVER`: The certificate resolver for the Traefik dashboard.
- `TIMEZONE`: The timezone for the server.
- `DOCKER_GID`: The group ID of the Docker group. See the instructions below to get the group ID.
- `TRAEFIK_UID`: The user ID of the traefik user.
- `SECRETS_PATH_CF_API_KEY`: The path to the Cloudflare API key (optional).
- `SECRETS_PATH_CF_EMAIL`: The path to the Cloudflare email address (optional).


Run the following command to read the environment variables from the `.env` file. This step is recommended to skip changing the following commands.
```shell
source .env
```

### Create a traefik user to run traefik in rootless mode

Create a new traefik user on the server to run the traefik container as a non-root user.
You can change the user id to any other value, but make sure that the user id is not already in use, and you change the `TRAEFIK_UID` in the `.env` file.
Change the ownership of the traefik directory to the new user and set the permissions of the acme.json file.

```shell
sudo useradd -u $TRAEFIK_UID -M -s /usr/sbin/nologin traefik
```

#### Create Base-Structure for traefik

Create the base structure for traefik and set the permissions.

```shell
mkdir -p $CERT_PATH
mkdir -p $HTPASSWD_PATH
mkdir -p $TRAEFIK_STORAGE_PATH/logs
mkdir -p $ACME_STORAGE_PATH
touch $TRAEFIK_STORAGE_PATH/logs/access.log
touch $TRAEFIK_STORAGE_PATH/logs/traefik.log
touch $ACME_STORAGE_PATH/acme.json
touch $ACME_STORAGE_PATH/cloudflare.json

chown -R $TRAEFIK_UID:$TRAEFIK_UID $TRAEFIK_STORAGE_PATH
chown -R $TRAEFIK_UID:$TRAEFIK_UID $ACME_STORAGE_PATH
chmod 600 $ACME_STORAGE_PATH/*.json
```

### Docker Group-ID

Traefik does not access the docker socket directly, instead it uses a docker-proxy to access the docker socket.
The proxy needs the correct docker group.

Get the GID if the docker group and write it into the `.env`-file with the following command
```shell
DOCKER_GID=$(grep '^docker:' /etc/group | cut -d':' -f3)
grep -q '^DOCKER_GID=' .env && sed -i "s/^DOCKER_GID=.*/DOCKER_GID=$DOCKER_GID/" .env || echo "DOCKER_GID=$DOCKER_GID" >> .env
```

### Start the containers

```shell
docker compose up -d
```


## Traefik

### Usage

To use this Traefik reverse proxy, configure your web services to include the appropriate labels in their Docker Compose files.
Consult the Traefik documentation for details on how to configure routing and SSL certificates.

```yaml
services:
  whoami:
    image: traefik/whoami
    networks:
      - traefik
    labels:
      - "traefik.enable=true"
      ## HTTP Routers
      - "traefik.http.routers.whoami-rtr.rule=Host(`whoami.$ROOT_DOMAIN_NAME`)"
      - "traefik.http.routers.whoami-rtr.entrypoints=websecure"
      - "traefik.http.routers.whoami-rtr.tls=true"

networks:
  traefik:
    external: true
```

### Add a new certificate

To add a new certificate, add the path to the `certs.yaml file`. Traefik loads them automatically.

The path `/etc/traefik/certs/` must be kept. Just change the filename to the corresponding filename of the cert files.

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

Traefik will automatically reload the certificates if you change the file. If you replace an existing certificate file, you may need to add/remove a line break to/from the file to trigger the reload.

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

### Logrotate - Traefik access log

Traefik does not support an automatic log rotation. You have to use logrotate for this.

Check if logrotate is installed.

```shell
logrotate --version
```

Create a new logrotate configuration file for Traefik.

```shell
echo "${TRAEFIK_STORAGE_PATH}/logs/access.log {
  daily
  rotate 10
  compress
  missingok
  notifempty
  postrotate
    docker kill --signal="USR1" traefik
  endscript
}" | sudo tee /etc/logrotate.d/traefik > /dev/null
```

#### Options

- `daily` → rotate daily.
- `rotate 10` → keep 10 logs.
- `compress` → compress the rotated logs with gzip.
- `missingok` → if the log file is missing, go on to the next one.
- `notifempty` → do not rotate the log if it is empty.
- `postrotate` → start of the postrotate script.
- `docker kill --signal="USR1" traefik` → send a signal to the Traefik container to close and reopen the log file. This requires the `traefik` container name.
- `endscript` → end of the postrotate script.

#### Testing the logrotate

To test the logrotate, you can run the following command:

```shell
sudo logrotate -d /etc/logrotate.d/traefik
```

This will run the logrotate in debug mode and show you what it would do. If everything looks good, you can run the logrotate without the `-d` flag.

```shell
sudo logrotate -f /etc/logrotate.d/traefik
```

### Optional Cloudflare Certificate Resolver

You have to follow the upper instructions first.

To use the Cloudflare integration, you need to add your Cloudflare API credentials to the secret files that you have defined in the `.env` file.

- `SECRETS_PATH_CF_API_KEY` contains the file path for the api key
- `SECRETS_PATH_CF_EMAIL` contains the file path for your email address

**Make sure that there is no empty line at the end of the secret files!**

Set your email address in the [traefik.yml](AppData/traefik-proxy/traefik.yml) file.
```yaml
certificatesResolvers:
   dns-cloudflare:
      acme:
         # ToDo: Change this value with your email address
         email: 'your@mail.com'
```

Change the cert resolver for the dashboard in the `.env` file.

```text
TRAEFIK_DASHBOARD_CERT_RESOLVER=dns-cloudflare
```


## Optional Features / Integrations

### Authentication Server

If you wish to use an authentication server for user authentication, refer to the documentation of [this repository](https://github.com/erkenes/docker-authentik) for setup instructions.


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
