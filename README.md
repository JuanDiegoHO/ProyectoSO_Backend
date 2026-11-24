# Proyecto final Corte 3 · Backend en Azure con Docker

Sistemas Operativos 103027 · Universidad Autónoma de Manizales

Parte 2 del trabajo final: una API en Flask y una base de datos PostgreSQL que corren en una máquina virtual de Azure con Docker Compose. La API entrega la lista de sistemas operativos que muestra el frontend, que vive en AWS EC2 y está en otro repositorio.

| | |
|---|---|
| API | `https://api-gNN.<dominio-del-curso>` *(pendiente de subdominio)* |
| Frontend | `https://gNN.<dominio-del-curso>` · repositorio: *(enlace al repo del frontend)* |
| Imagen en Docker Hub | [`yexg/so-backend:1.0`](https://hub.docker.com/r/yexg/so-backend) |
| Integrantes | *(nombres)* |

## Arquitectura

```
Navegador ──► https://api-gNN.<dominio>  (VM Ubuntu en Azure)
                 │
                 ▼
              Nginx :443  (certificado de Let's Encrypt)
                 │  proxy_pass
                 ▼
              contenedor backend  127.0.0.1:8000  (gunicorn + Flask)
                 │  red interna de Compose
                 ▼
              contenedor db  (PostgreSQL 16, sin puertos publicados)
```

- Solo Nginx recibe tráfico de internet (puertos 80 y 443).
- El backend escucha únicamente en `127.0.0.1:8000`, así que desde fuera solo se llega a él a través de Nginx.
- La base de datos no publica ningún puerto: solo el backend la alcanza por la red interna de Compose, usando el nombre de servicio `db` como host.

## Estructura

```
backend/
  app.py            API en Flask: /api/salud y /api/sistemas (código del docente)
  requirements.txt  dependencias (Flask, psycopg, gunicorn)
  Dockerfile        imagen del backend
  .dockerignore     deja fuera de la imagen .venv, cachés y .env
  .env.example      variables que necesita el despliegue
db/
  init.sql          crea la tabla y carga los datos (código del docente)
compose.yaml        servicios db y backend para la VM de Azure
nginx/
  azure-api.conf    proxy inverso de Nginx para el subdominio de la API
```

## Endpoints

| Ruta | Respuesta |
|---|---|
| `GET /api/salud` | `{"estado": "ok"}` si la base de datos contesta |
| `GET /api/sistemas` | lista de sistemas operativos guardada en la base de datos |

## 1. La imagen del backend

El `Dockerfile` parte de `python:3.12-slim` (versión fija, nunca `latest`), instala `requirements.txt`, copia `app.py`, corre con un usuario sin privilegios y arranca la API con gunicorn en `0.0.0.0:8000` sirviendo `app:app`.

La imagen se construye en el PC y se publica en Docker Hub; nunca se construye en la VM.

```bash
docker build -t yexg/so-backend:1.0 ./backend
docker push yexg/so-backend:1.0
```

> En un Mac con chip M hay que agregar `--platform linux/amd64` al `docker build`, porque la VM es x86_64.

## 2. La máquina virtual en Azure

- Suscripción **Azure for Students** (sin tarjeta).
- Imagen **Ubuntu Server 24.04 LTS**, autenticación por **llave SSH**, usuario `azureuser`.
- Grupo de seguridad de red con los puertos **22, 80 y 443** abiertos, nada más.
- Región **Brazil South** y tamaño **Standard D2s v4** (2 vCPU, 8 GiB). Ver la sección de problemas: los tamaños más pequeños no estaban disponibles para la suscripción de estudiante.

Conexión:

```bash
ssh -i ~/.ssh/<llave> azureuser@<IP_DE_AZURE>
```

Instalación de Docker en la VM:

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker azureuser
exit   # volver a entrar para que aplique el grupo docker
docker --version && docker compose version
```

## 3. Levantar la base de datos y la API con Compose

En la VM:

```bash
git clone https://github.com/JuanDiegoHO/ProyectoSO_Backend so
cd so
cp backend/.env.example .env
nano .env
```

En `.env` se pone una clave propia (solo letras y números, para no romper la URL) y la misma en `POSTGRES_PASSWORD` y en `DATABASE_URL`. El host de la base dentro de la red de Compose es `db`, el nombre del servicio:

```
POSTGRES_USER=so
POSTGRES_PASSWORD=<clave>
POSTGRES_DB=so
DATABASE_URL=postgresql://so:<clave>@db:5432/so
ORIGEN_PERMITIDO=https://gNN.<dominio-del-curso>
```

El `.env` real no se sube a GitHub (está en `.gitignore`).

```bash
docker compose up -d
docker compose ps
curl http://localhost:8000/api/sistemas
```

`compose.yaml` define:

- **db**: `postgres:16`, usuario, clave y base tomados del `.env`, un volumen `pgdata` para los datos y `db/init.sql` montado en `/docker-entrypoint-initdb.d/` para que cree la tabla y cargue los datos al iniciar. Sin puertos publicados.
- **backend**: la imagen de Docker Hub (con `image:`, no `build:`), `DATABASE_URL` y `ORIGEN_PERMITIDO` desde el `.env`, arranca después de `db` y se reinicia solo (`restart: unless-stopped`).

> `init.sql` solo se ejecuta la primera vez, con el volumen vacío. Para volver a cargarlo (o si se cambia la clave de la base después del primer arranque) hay que borrar el volumen con `docker compose down -v`.

## 4. Proxy inverso y HTTPS

```bash
sudo apt update && sudo apt install -y nginx certbot python3-certbot-nginx
sudo cp nginx/azure-api.conf /etc/nginx/sites-available/api
sudo nano /etc/nginx/sites-available/api      # server_name = api-gNN.<dominio>
sudo ln -s /etc/nginx/sites-available/api /etc/nginx/sites-enabled/api
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

Cuando `nslookup api-gNN.<dominio>` devuelve la IP de la VM:

```bash
sudo certbot --nginx -d api-gNN.<dominio>
curl https://api-gNN.<dominio>/api/sistemas
```

## 5. Integración con el frontend

1. En el frontend (EC2), `config.js` apunta a `https://api-gNN.<dominio>`.
2. En la VM, `ORIGEN_PERMITIDO` en `.env` es exactamente la URL del frontend (`https://gNN.<dominio>`, sin `/` al final), y se recrea el backend:

   ```bash
   docker compose up -d
   ```

3. `https://gNN.<dominio>` muestra la tabla llena. En el navegador (F12 → Network) se ve que es el **navegador del usuario** quien hace la petición a la API, no el servidor de EC2.

## Publicar una nueva versión

```bash
# en el PC
docker build -t yexg/so-backend:1.1 ./backend
docker push yexg/so-backend:1.1

# en la VM: cambiar 1.0 por 1.1 en compose.yaml y luego
docker compose pull
docker compose up -d
```

## Problemas que encontramos

| Síntoma | Causa y solución |
|---|---|
| `NotAvailableForSubscription` al elegir B1s u otros tamaños pequeños | La suscripción Azure for Students no tenía capacidad para esos tamaños en las regiones permitidas. Se usó **D2s v4** en Brazil South, que sí estaba disponible. |
| `RequestDisallowedByAzure` al validar la VM | La suscripción tiene una política (*Allowed resource deployment regions*) que solo permite algunas regiones. Se revisó en **Directiva → Asignaciones** y se eligió una región permitida (Brazil South). |
| `failed to connect to the docker API ... dockerDesktopLinuxEngine` o error 500 en `_ping` al hacer `docker build` | Docker Desktop no estaba arrancado. Se abrió (o reinició) y se esperó a ver *Engine running*. |
| La tabla no existe | `init.sql` solo corre con el volumen vacío: `docker compose down -v` y `docker compose up -d`. |
| 502 Bad Gateway en la API | El backend no está corriendo o no escucha en el puerto 8000: `docker compose ps` y `docker compose logs backend`. |
| Error de CORS en el navegador | `ORIGEN_PERMITIDO` no coincide exactamente con la URL del frontend. |

## Costos y limpieza

- La VM gasta crédito de Azure for Students mientras está encendida. Cuando no se usa se **detiene** desde el portal (la IP pública es estática y los contenedores vuelven a arrancar solos).
- Al terminar el curso se borra el **grupo de recursos completo** (`so-corte3`) desde el portal de Azure.
