# Guía: Mattermost

---

## PASO 1: Preparar el servidor Ubuntu

```bash
# Actualizacion de sistema
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget nano ufw

# Para permitir el SSH y el puerto de Mattermost
sudo ufw allow 22/tcp
sudo ufw allow 8065/tcp
sudo ufw enable
sudo ufw status
```

> Habilitamos el SSH para poder conectarnor y poder trabajar de forma mas comoda y el puerto 8065 porque es donde trabaja Mattermost.
> 

---

## PASO 2: Instalar Docker

```bash
# Instalar Docker
curl -fsSL https://get.docker.com | sudo sh

# Añadir usuario al grupo docker
sudo usermod -aG docker $USER
newgrp docker

# Instalar Docker Compose
sudo apt install -y docker-compose-plugin

# Verificar
docker --version
docker compose version
```

---

## PASO 3: Crear estructura de carpetas

```bash
mkdir -p ~/mattermost
cd ~/mattermost

mkdir -p ./volumes/app/mattermost/{config,data,logs,plugins,client/plugins,bleve-indexes}
mkdir -p ./volumes/db
```

---

## PASO 4: Crear el `docker-compose.yml`

```bash
nano ~/mattermost/docker-compose.yml
```

Contenido:

```yaml
version: "3.9"

services:

  postgres:
    image: postgres:15-alpine
    container_name: mattermost-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: mmuser
      POSTGRES_PASSWORD: mmpassword_segura_2024
      POSTGRES_DB: mattermost
    volumes:
      - ./volumes/db:/var/lib/postgresql/data
    networks:
      - mattermost-net
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "mmuser", "-d", "mattermost"]
      interval: 10s
      timeout: 5s
      retries: 5

  mattermost:
    image: mattermost/mattermost-team-edition:latest
    container_name: mattermost-app
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      MM_SQLSETTINGS_DRIVERNAME: postgres
      MM_SQLSETTINGS_DATASOURCE: postgres://mmuser:mmpassword_segura_2024@postgres:5432/mattermost?sslmode=disable&connect_timeout=10
      MM_BLEVESETTINGS_INDEXDIR: /mattermost/bleve-indexes
      MM_SERVICESETTINGS_SITEURL: http://192.168.109.155:8065
    volumes:
      - ./volumes/app/mattermost/config:/mattermost/config:rw
      - ./volumes/app/mattermost/data:/mattermost/data:rw
      - ./volumes/app/mattermost/logs:/mattermost/logs:rw
      - ./volumes/app/mattermost/plugins:/mattermost/plugins:rw
      - ./volumes/app/mattermost/client/plugins:/mattermost/client/plugins:rw
      - ./volumes/app/mattermost/bleve-indexes:/mattermost/bleve-indexes:rw
    ports:
      - "8065:8065"
    networks:
      - mattermost-net

networks:
  mattermost-net:
    driver: bridge
```

---

## PASO 5: Ajustar permisos y arrancar

```bash
# Mattermost usa internamente el UID 2000
sudo chown -R 2000:2000 ~/mattermost/volumes/app/mattermost

# Levantar todo
cd ~/mattermost
docker compose up -d

# Verificar que los contenedores están corriendo
docker compose ps

# Ver logs en vivo
docker compose logs -f mattermost
```

## PASO 6: Primer acceso y registro de admin

1. Clic en **"Create Account"**
2. Rellena email, usuario y contraseña
3. **El primer usuario registrado es automáticamente administrador**
4. Dale nombre a tu servidor y crea un equipo

> `http://192.168.109.155:8065`
> 

---

## PASO 7: Permitir que cualquiera se registre

1. Entra a: **System Console**
2. Ve a **Authentication → Email**
    - *Enable account creation* = **true**
3. Ve a **Site Configuration → Users and Teams**
    - *Enable Open Server* = **true** ( esto permite registrarnos sin invitación )

---

## Comandos útiles del día a día

```bash
cd ~/mattermost

docker compose down          # Apagar
docker compose up -d         # Arrancar
docker compose restart       # Reiniciar
docker compose logs -f       # Ver logs

# Actualizar
docker compose pull
docker compose up -d

# Backup de la base de datos
docker exec mattermost-db pg_dump -U mmuser mattermost > backup_$(date +%Y%m%d).sql
```