[Documentación WikiJS](https://docs.requarks.io/install/ubuntu)
## Paso 1: Actualiza la Máquina

```
# Fetch latest updates
sudo apt -qqy update

# Install all updates automatically
sudo DEBIAN_FRONTEND=noninteractive apt-get -qqy -o Dpkg::Options::='--force-confdef' -o Dpkg::Options::='--force-confold' dist-upgrade
```
## Paso 2: Instale Docker

```
# Install dependencies to install Docker
sudo apt -qqy -o Dpkg::Options::='--force-confdef' -o Dpkg::Options::='--force-confold' install ca-certificates curl gnupg lsb-release

# Register Docker package registry
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --imprimir-arquitectura) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Refresh package udpates and install Docker
sudo apt -qqy update
sudo apt -qqy -o Dpkg::Options::='--force-confdef' -o Dpkg::Options::='--force-confold' install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
## Paso 3: Configurar Contenedores

```
# Create installation directory for Wiki.js
mkdir -p /etc/wiki

# Generate DB secret
openssl rand -base64 32 > /etc/wiki/.db-secret

# Create internal docker network
docker network create wikinet

# Create data volume for PostgreSQL
docker volume create pgdata

# Create the containers
docker create --name=db -e POSTGRES_DB=wiki -e POSTGRES_USER=wiki -e POSTGRES_PASSWORD_FILE=/etc/wiki/.db-secret -v /etc/wiki/.db-secret:/etc/wiki/.db-secret:ro -v pgdata:/var/lib/postgresql/data --restart=unless-stopped -h db --network=wikinet postgres:15
docker create --name=wiki -e DB_TYPE=postgres -e DB_HOST=db -e DB_PORT=5432 -e DB_PASS_FILE=/etc/wiki/.db-secret -v /etc/wiki/.db-secret:/etc/wiki/.db-secret:ro -e DB_USER=wiki -e DB_NAME=wiki -e UPGRADE_COMPANION=1 --restart=unless-stopped -h wiki --network=wikinet -p 80:3000 -p 443:3443 ghcr.io/requarks/wiki:2
docker create --name=wiki-update-companion -v /var/run/docker.sock:/var/run/docker.sock:ro --restart=unless-stopped -h wiki-update-companion --network=wikinet ghcr.io/requarks/wiki-update-companion:latest
```

## Paso 4: Configurar Firewall

```
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https

sudo ufw --force enable
```

## Paso 5: Inicie los Contenedores

```
docker start db
docker start wiki
docker start wiki-update-companion
```