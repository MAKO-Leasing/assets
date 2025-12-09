# Guía Completa: Despliegue de Aplicación Node.js en VPS Hostinger

## 1. Configuración DNS en GoDaddy

1. Accede al panel de GoDaddy y ve a "Mis Productos" → "DNS"
2. Agrega un registro tipo **A**:
   - **Nombre**: `api` (o el subdominio que desees)
   - **Valor**: IP de tu VPS Hostinger
   - **TTL**: 600 segundos (o el predeterminado)
3. Guarda los cambios
4. **Espera** 5-10 minutos para que se propague el DNS

---

## 2. Preparar el Servidor (Dependencias)

```bash
# Actualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar Node.js (versión LTS)
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs

# Verificar instalación
node --version
npm --version

# Instalar PM2 globalmente
sudo npm install -g pm2

# Instalar Nginx
sudo apt install -y nginx

# Instalar Git (si no está instalado)
sudo apt install -y git
```

---

## 3. Crear Estructura de Directorios

```bash
# Crear directorio del proyecto
sudo mkdir -p /var/www/api.mako.mx

# Dar permisos al usuario actual (reemplaza 'usuario' con tu usuario)
sudo chown -R $USER:$USER /var/www/api.mako.mx
```

---

## 4. Clonar y Configurar el Proyecto

```bash
# Ir al directorio
cd /var/www/api.mako.mx

# Clonar el repositorio (con credenciales)
git clone https://miguelmhz:ghp_xxxxxxbkKYPFQ6xWLK1vdxu2@github.com/MAKO-Leasing/mako-plataforma-backend.git .

# O si ya clonaste sin credenciales, actualizar remote
git remote set-url origin https://miguelmhz:ghp_xxxxxxbkKYPFQ6xWLK1vdxu2@github.com/MAKO-Leasing/mako-plataforma-backend.git

# Instalar dependencias
npm install

# Compilar TypeScript
npm run build

# Crear archivo .env si es necesario
nano .env
```

---

## 5. Configurar PM2

```bash
# Iniciar aplicación con PM2 (ajusta el comando según tu proyecto)
pm2 start dist/index.js --name api

# O si usas npm script:
pm2 start npm --name api -- start

#un puerto en particular (next)
PORT=3011 pm2 start pnpm --name plataforma-dev.mako.mx -- start

# Guardar configuración de PM2
pm2 save

# Configurar PM2 para iniciar al reiniciar servidor
pm2 startup
# Ejecuta el comando que PM2 te muestre
```

---

## 6. Configurar Nginx (SIN SSL primero)

```bash
# Crear archivo de configuración
sudo nano /etc/nginx/sites-available/api.mako.mx
```

**Contenido del archivo** (reemplaza `3000` con el puerto de tu app):

```nginx
server {
    listen 80;
    server_name api.mako.mx;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## 7. Activar Sitio y Verificar Nginx

```bash
# Crear enlace simbólico
sudo ln -s /etc/nginx/sites-available/api.mako.mx /etc/nginx/sites-enabled/

# Verificar sintaxis de Nginx
sudo nginx -t

# Si todo está bien, recargar Nginx
sudo systemctl reload nginx

# Verificar estado
sudo systemctl status nginx
```

**Prueba en el navegador**: `http://api.mako.mx` (debería funcionar)

---

## 8. Configurar SSL con Certbot

```bash
# Instalar Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtener e instalar certificado SSL automáticamente
sudo certbot --nginx -d api.mako.mx

# Seguir las instrucciones (ingresa tu email, acepta términos)
```

Certbot modificará automáticamente tu configuración de Nginx para:
- Redirigir HTTP a HTTPS
- Configurar el certificado SSL

```bash
# Verificar renovación automática
sudo certbot renew --dry-run
```

---

## 9. Script de Automatización de Deploy

```bash
# Crear script
sudo nano /var/www/deploy-api.sh
```

**Contenido** (con correcciones):

```bash
#!/bin/bash
set -e  # Detener si hay algún error

# Configuración
REPO_DIR="/var/www/api.mako.mx"
BRANCH="main"
PM2_APP_NAME="api"
LOG_FILE="/var/log/deploy-$(date +%Y%m%d-%H%M%S).log"

# Crear directorio de logs si no existe
mkdir -p /var/log

# Función para logging
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "-----------------------------------------"
log "Iniciando deployment"
log "Repositorio: $REPO_DIR"
log "Rama: $BRANCH"
log "-----------------------------------------"

# Cambiar al directorio del repositorio
cd "$REPO_DIR" || { log "ERROR: No se encontró el directorio $REPO_DIR"; exit 1; }

# Hacer fetch de todas las ramas
log "Obteniendo ramas remotas..."
git fetch origin

# Verificar si la rama existe localmente
if git show-ref --verify --quiet "refs/heads/$BRANCH"; then
    log "Rama $BRANCH ya existe localmente"
    git checkout "$BRANCH"
else
    log "Creando rama local $BRANCH desde origin/$BRANCH"
    git checkout -b "$BRANCH" "origin/$BRANCH"
fi

# Obtener el hash antes del pull
BEFORE_HASH=$(git rev-parse HEAD)

# Ejecutar git pull
log "Ejecutando git pull..."
git pull origin "$BRANCH"

# Obtener el hash después del pull
AFTER_HASH=$(git rev-parse HEAD)

# Verificar si hubo cambios
if [ "$BEFORE_HASH" = "$AFTER_HASH" ]; then
    log "No hay cambios nuevos. Deployment cancelado."
else
    log "Cambios detectados: $BEFORE_HASH -> $AFTER_HASH"
    
    # Verificar si package.json cambió
    if git diff --name-only "$BEFORE_HASH" "$AFTER_HASH" | grep -q "package.json"; then
        log "Detectados cambios en dependencias. Ejecutando npm install..."
        npm install
    else
        log "Sin cambios en dependencias. Saltando npm install."
    fi
    
    log "Compilando TypeScript..."
    npm run build
    
    log "Recargando proceso PM2: $PM2_APP_NAME"
    pm2 reload "$PM2_APP_NAME" --update-env
    
    # Verificar estado de PM2
    if pm2 describe "$PM2_APP_NAME" | grep -q "online"; then
        log "✅ Proceso recargado exitosamente"
    else
        log "❌ ERROR: El proceso no está online"
        pm2 logs "$PM2_APP_NAME" --lines 20
        exit 1
    fi
fi

log "-----------------------------------------"
log "Deployment finalizado exitosamente"
log "Log guardado en: $LOG_FILE"
log "-----------------------------------------"
```

```bash
# Dar permisos de ejecución
sudo chmod +x /var/www/deploy-api.sh

# Probar el script
sudo /var/www/deploy-api.sh
```

---

## 10. GitHub Action (Opcional)

```bash
# En tu repositorio, crear: .github/workflows/deploy.yml
```

```yaml
name: Deploy to Main VPS

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          passphrase: ${{ secrets.VPS_SSH_PASSPHRASE }}
          port: ${{ secrets.VPS_PORT }}
          script: |
            bash /var/www/update-api-plataforma.sh
```

**Configurar Secrets en GitHub**:
1. Ve a tu repositorio → Settings → Secrets → Actions
2. Agrega:
   - `VPS_HOST`: srv1089405.hstgr.cloud
   - `VPS_USERNAME`: root
   - `VPS_SSH_KEY`: -----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABBTzlJ3UH
k89PmZiS2eVu1hAAAAGAAAAAEAAAAzAAAAC3NzaC1lZDI1NTE5AAAAID9mydceond3xYxY
b/oi7mzjGljNWzNgZnKb25SW8rjUAAAAoNgchDp/ZbeA5vDMzQtTmPcdDOpnjRVrtvNslX
1y5uCn87lmrwLa6IhGV/XcpqeZjszhfjKwN+/Vy8fus9QGBzxPUXOj6VcXcc+3vkBfUTHB
bzLYfbSAPPT3wVYH+4K1B+YpF8q9WW9PLu8jyaR2iSlbDPr6Baqi7svT9wiN2B2Z58e7Ma
bzvCD5CuBuK7jN1aPWQWtCRQDESEpNvTyl68k=
-----END OPENSSH PRIVATE KEY-----
   - `VPS_SSH_PASSPHRASE`: makocapital
  



Mike
  10:51
puerto: 22

**Para generar y copiar SSH key al VPS**:
```bash
# En tu máquina local o en GitHub Actions
ssh-keygen -t ed25519 -C "github-actions"

# Copiar key pública al VPS
ssh-copy-id usuario@ip-vps

# O manualmente:
cat ~/.ssh/id_ed25519.pub
# Copiar y pegar en VPS: ~/.ssh/authorized_keys
```

---

## 11. Comandos Útiles

```bash
# Ver logs de PM2
pm2 logs api

# Ver estado de procesos
pm2 status

# Reiniciar app
pm2 restart api

# Ver logs de Nginx
sudo tail -f /var/nginx/error.log

# Ver logs de deployment
sudo tail -f /var/log/deploy-*.log

# Verificar puerto en uso
sudo netstat -tulpn | grep :3000
```

---



En el servidor PostgreSQL (217.196.51.110):
bash# Conectarse al servidor de PostgreSQL
ssh user@217.196.51.110

# Editar pg_hba.conf
sudo nano /etc/postgresql/*/main/pg_hba.conf

# O buscar la ubicación exacta
sudo find /etc/postgresql -name pg_hba.conf
Agregar esta línea al final del archivo:
conf# Permitir conexión desde tu servidor de aplicación
host    extractor-mako    mako    189.183.106.142/32    md5

