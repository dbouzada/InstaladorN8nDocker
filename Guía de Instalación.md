# n8n en Docker — Guía de instalación

Guía paso a paso para autoalojar n8n usando Docker y Docker Compose en un VPS con Linux (Ubuntu 22.04+).

---

## Cómo usar esta guía

Todos los comandos se ejecutan **en el servidor VPS**, conectado vía SSH desde tu terminal local. No se corre nada en tu computadora personal (salvo la conexión SSH inicial).

```
Tu computadora  ──SSH──▶  VPS (Ubuntu)  ──Docker──▶  n8n
```

---

## Requisitos previos

- VPS con Ubuntu 22.04 o superior
- Acceso SSH con usuario con permisos `sudo`
- Un dominio apuntando a la IP del VPS (necesario para SSL en el Paso 5)
- Puerto 80 y 443 abiertos en el firewall del VPS

### Conectarse al VPS por SSH

Desde tu terminal local (Mac/Linux) o PowerShell (Windows):

```bash
# Reemplazar usuario e IP con los tuyos
ssh usuario@IP_DEL_VPS
```

A partir de acá, **todos los comandos siguientes se ejecutan dentro de esa sesión SSH**, es decir, en el servidor remoto.

---

## Paso 1 — Verificar o instalar Docker

> 📍 **Dónde:** En el servidor, dentro de la sesión SSH.

Verificar si Docker ya está instalado:

```bash
docker -v
```

Si devuelve algo como `Docker version 26.1.0` podés pasar al Paso 2.

Si no está instalado, ejecutar el script oficial:

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Confirmar que quedó instalado:

```bash
docker -v
sudo docker run hello-world
```

> **Nota:** Desde Docker v20.10, `docker compose` viene incluido como plugin. No hace falta instalar Docker Compose por separado.

> **Tip:** Algunos VPS (como Hostinger Docker VPS) incluyen Docker preinstalado. En ese caso, saltar directo al Paso 2.

---

## Paso 2 — Crear el directorio de trabajo

> 📍 **Dónde:** En el servidor, dentro de la sesión SSH (podés estar en cualquier ubicación, el comando usa `~` que es el home del usuario).

Crear la carpeta del proyecto y moverse a ella:

```bash
mkdir ~/n8n && cd ~/n8n
```

Crear el subdirectorio donde n8n va a guardar sus datos (flujos, credenciales, historial de ejecuciones):

```bash
mkdir n8n_data
```

Asignar los permisos correctos. n8n corre internamente con el usuario `1000` dentro del contenedor y necesita poder escribir en esta carpeta:

```bash
sudo chown -R 1000:1000 n8n_data
```

Verificar que quedó bien:

```bash
ls -la ~/n8n
# Debería mostrar n8n_data con owner 1000:1000
```

---

## Paso 3 — Crear el archivo docker-compose.yml

> 📍 **Dónde:** En el servidor, dentro de la sesión SSH, en la carpeta `~/n8n`.

Este archivo le dice a Docker cómo levantar n8n: qué imagen usar, qué puertos exponer, qué variables de configuración aplicar y dónde guardar los datos.

Abrir el editor `nano` para crear el archivo:

```bash
nano docker-compose.yml
```

Dentro del editor, pegar el siguiente contenido. Reemplazar `tudominio.com` y la `N8N_ENCRYPTION_KEY` con tus propios valores:

```yaml
services:
  n8n:
    image: n8nio/n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=tudominio.com               # Reemplazar con tu dominio real
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://tudominio.com    # Reemplazar con tu dominio real
      - TZ=America/Argentina/Buenos_Aires
      - N8N_ENCRYPTION_KEY=cambiarPorUnaClaveLargaYAleatoria
    volumes:
      - ./n8n_data:/home/node/.n8n
```

Guardar el archivo y salir de nano:

```
Ctrl + X  →  Y  →  Enter
```

Confirmar que el archivo se creó correctamente:

```bash
cat docker-compose.yml
```

> **Nota:** La variable `N8N_BASIC_AUTH_ACTIVE` fue deprecada en n8n v1+. La autenticación se configura desde la interfaz web al registrar la cuenta "owner" en el primer acceso.

---

## Paso 4 — Levantar el contenedor

> 📍 **Dónde:** En el servidor, dentro de la sesión SSH, en la carpeta `~/n8n`.

Iniciar n8n en segundo plano:

```bash
docker compose up -d
```

Docker descarga la imagen de n8n (puede tardar unos minutos la primera vez) y luego inicia el contenedor. Deberías ver:

```
✔ Container n8n  Started
```

Verificar que el contenedor está corriendo:

```bash
docker compose ps
# El estado debe figurar como "Up" o "running"
```

Ver los logs para confirmar que n8n arrancó correctamente:

```bash
docker compose logs -f n8n
# Salir con Ctrl + C
```

Probar el acceso desde el navegador de tu computadora (aún sin SSL):

```
http://IP_DEL_VPS:5678
```

> Si aparece una advertencia de "Secure Cookie", es normal en este punto. Se resuelve en el Paso 5 al agregar HTTPS.

---

## Paso 5 — Configurar SSL con NGINX y Certbot

> 📍 **Dónde:** En el servidor, dentro de la sesión SSH. El dominio ya debe estar apuntando a la IP del VPS antes de este paso.

n8n necesita correr detrás de un proxy inverso con HTTPS para funcionar correctamente en producción. Usamos NGINX como proxy y Certbot para el certificado SSL gratuito de Let's Encrypt.

### 5.1 — Instalar NGINX y Certbot

```bash
sudo apt update
sudo apt install nginx certbot -y
```

Detener NGINX temporalmente para liberar el puerto 80 (Certbot lo necesita para verificar el dominio):

```bash
sudo systemctl stop nginx
```

### 5.2 — Obtener el certificado SSL

```bash
# Reemplazar tudominio.com con tu dominio real
sudo certbot certonly --standalone -d tudominio.com
```

Certbot va a pedir:
1. Un email de contacto
2. Aceptar los términos de servicio → `Y`
3. Suscribirse a noticias → opcional, podés poner `N`

Si todo va bien, el certificado queda en:

```
/etc/letsencrypt/live/tudominio.com/fullchain.pem
/etc/letsencrypt/live/tudominio.com/privkey.pem
```

### 5.3 — Configurar el proxy inverso

Crear el archivo de configuración de NGINX para n8n:

```bash
sudo nano /etc/nginx/sites-available/n8n
```

Pegar el siguiente contenido (reemplazar `tudominio.com` en todas las líneas donde aparece):

```nginx
server {
    listen 443 ssl;
    server_name tudominio.com;

    ssl_certificate /etc/letsencrypt/live/tudominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tudominio.com/privkey.pem;

    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Soporte para WebSockets — requerido por n8n
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

# Redirigir HTTP a HTTPS automáticamente
server {
    listen 80;
    server_name tudominio.com;
    return 301 https://$host$request_uri;
}
```

Guardar y salir: `Ctrl + X` → `Y` → `Enter`

### 5.4 — Activar y reiniciar NGINX

Activar la configuración creando un enlace simbólico:

```bash
sudo ln -sf /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/n8n
```

Verificar que no hay errores de sintaxis:

```bash
sudo nginx -t
# Debe decir: configuration file ... syntax is ok
```

Iniciar y recargar NGINX:

```bash
sudo systemctl start nginx
sudo systemctl reload nginx
```

---

## Paso 6 — Acceder a n8n

> 📍 **Dónde:** Desde el navegador de tu computadora local (no en el servidor).

Abrir en el navegador:

```
https://tudominio.com
```

Deberías ver la pantalla de registro de n8n. Completar los datos para crear la cuenta "owner". Esta es la cuenta de administrador principal con la que vas a gestionar todos los flujos y credenciales.

---

## Paso 7 — Ajustar variables de entorno (opcional)

> 📍 **Dónde:** En el servidor, dentro de la sesión SSH, en la carpeta `~/n8n`.

Si necesitás cambiar la configuración de n8n, editar el archivo:

```bash
nano ~/n8n/docker-compose.yml
```

Variables útiles para agregar en la sección `environment`:

```yaml
environment:
  # Base de datos PostgreSQL (recomendado para producción en lugar del SQLite por defecto)
  - DB_TYPE=postgresdb
  - DB_POSTGRESDB_HOST=localhost
  - DB_POSTGRESDB_PORT=5432
  - DB_POSTGRESDB_DATABASE=n8n
  - DB_POSTGRESDB_USER=n8n_user
  - DB_POSTGRESDB_PASSWORD=contraseña_segura

  # Zona horaria
  - TZ=America/Argentina/Buenos_Aires

  # URL pública para webhooks
  - WEBHOOK_URL=https://tudominio.com

  # Clave de cifrado para credenciales almacenadas
  - N8N_ENCRYPTION_KEY=claveAleatoriaMuyLarga
```

Aplicar los cambios reiniciando el contenedor:

```bash
# Desde ~/n8n en el servidor
docker compose down
docker compose up -d
```

> Para reiniciar sin cambios de configuración: `docker compose restart`

---

## Paso 8 — Actualizar n8n

> 📍 **Dónde:** En el servidor, dentro de la sesión SSH, en la carpeta `~/n8n`.

Para actualizar n8n a la última versión:

```bash
# 1. Descargar la nueva imagen desde Docker Hub
docker pull n8nio/n8n

# 2. Recrear el contenedor con la imagen actualizada (los datos se mantienen)
docker compose up -d
```

Docker reemplaza el contenedor automáticamente. Todos los datos en `n8n_data` quedan intactos.

---

## Buenas prácticas

| Práctica | Detalle |
|---|---|
| **Volúmenes** | Siempre montar `./n8n_data:/home/node/.n8n` para persistir datos entre reinicios |
| **Backups** | Copiar la carpeta `n8n_data` periódicamente (ver comando abajo) |
| **SSL** | Nunca exponer n8n sin HTTPS en producción |
| **Firewall** | Cerrar el puerto 5678 al exterior; solo dejar abiertos 80, 443 y 22 |
| **Encryption key** | Definir `N8N_ENCRYPTION_KEY` antes del primer arranque y no cambiarla después |
| **Recursos** | Limitar CPU/RAM en el compose para evitar que n8n consuma todo el VPS |

### Cerrar el puerto 5678 con UFW

> 📍 **Dónde:** En el servidor, dentro de la sesión SSH.

```bash
sudo ufw deny 5678
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 22    # SSH — no bloquear o perdés el acceso remoto
sudo ufw enable
```

### Limitar recursos del contenedor (opcional)

Agregar dentro del servicio `n8n` en `docker-compose.yml`:

```yaml
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G
```

### Backup manual de los datos

> 📍 **Dónde:** En el servidor, dentro de la sesión SSH.

```bash
docker cp n8n:/home/node/.n8n ~/backup-n8n-$(date +%Y%m%d)
```

---

## Comandos de referencia rápida

> 📍 **Dónde:** Todos estos comandos se ejecutan en el servidor vía SSH, dentro de `~/n8n`.

```bash
# Ver estado del contenedor
docker compose ps

# Ver logs en tiempo real (salir con Ctrl+C)
docker compose logs -f n8n

# Detener n8n
docker compose down

# Reiniciar n8n (sin cambios de configuración)
docker compose restart

# Actualizar a la última versión
docker pull n8nio/n8n && docker compose up -d
```

---

## Referencias

- [Documentación oficial de n8n](https://docs.n8n.io/)
- [Variables de entorno de n8n](https://docs.n8n.io/hosting/environment-variables/)
- [Docker Hub — n8nio/n8n](https://hub.docker.com/r/n8nio/n8n)
- [Instalación de Docker en Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Certbot — Let's Encrypt](https://certbot.eff.org/)
