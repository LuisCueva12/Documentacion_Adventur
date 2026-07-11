# Despliegue — SiteGround a VPS

**Carpeta:** 06 — Arquitectura Técnica  
**Documento:** 05 — Despliegue y Migración

---

## 1. Estrategia de Despliegue

```
FASE 1 ──────────────────────►  FASE 2 ──────────────────────►  FASE 3
SiteGround                      VPS Propio                      Multi-agencia
(Coste bajo, inicio)            (Escalabilidad)                 (Crecimiento)
```

---

## 2. Fase 1 — SiteGround (Inicio)

### 2.1 Requisitos del Plan SiteGround

| Recurso | Mínimo |
|---------|--------|
| PHP | 8.4+ |
| PostgreSQL | 16+ (o MySQL como alternativa temporal) |
| Node.js | No necesario (frontend compilado) |
| Composer | Sí, vía SSH |
| SSL | SSL gratuito incluido (Let's Encrypt) |
| Almacenamiento | 10 GB+ |
| Dominio | `app.adventur.com` o `admin.adventur.com` |

### 2.2 Estructura en SiteGround

```
public_html/                     ← Raíz del sitio web
│
├── index.html                   ← Frontend compilado (SPA)
├── assets/                      ← JS, CSS, imágenes (compilado por Vite)
│   ├── index-abc123.js
│   └── index-xyz456.css
│
├── .htaccess                    ← Reglas de rewrite para SPA
│
└── api/                         ← Laravel (como subdirectorio)
    ├── public/
    │   └── index.php            ← Entry point Laravel
    ├── app/
    ├── config/
    └── ...
```

### 2.3 Configuración .htaccess (SiteGround)

```apache
# public_html/.htaccess
# Redirigir todo al index.html (SPA) excepto /api

<IfModule mod_rewrite.c>
    RewriteEngine On

    # No aplicar rewrite a /api (Laravel maneja sus propias rutas)
    RewriteRule ^api/ - [L]

    # No aplicar rewrite a archivos reales
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d

    # Todo lo demás → index.html (SPA)
    RewriteRule ^ /index.html [L]
</IfModule>
```

### 2.4 Archivo .env (SiteGround)

```env
APP_NAME="Adventur Travel"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://app.adventur.com

# API en subdirectorio /api
APP_URL_API=https://app.adventur.com/api

DB_CONNECTION=pgsql
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE=adventur_db
DB_USERNAME=adventur_user
DB_PASSWORD=********

# Sanctum - importante: frontend comparte mismo dominio
SANCTUM_STATEFUL_DOMAINS=app.adventur.com
SESSION_DRIVER=cookie
SESSION_DOMAIN=.adventur.com
```

### 2.5 Build y Deploy Manual

```bash
# 1. Compilar Frontend
cd adventur-frontend
npm install
npm run build
# → genera dist/

# 2. Subir frontend a SiteGround
rsync -avz dist/ usuario@siteground:public_html/

# 3. Subir backend
rsync -avz --exclude='vendor' --exclude='node_modules' \
  adventur-api/ usuario@siteground:public_html/api/

# 4. En SiteGround (SSH)
cd public_html/api
composer install --no-dev --optimize-autoloader
php artisan config:cache
php artisan route:cache
php artisan migrate

# 5. Permisos
chmod -R 775 storage bootstrap/cache
```

### 2.6 Limitaciones de SiteGround

| Aspecto | SiteGround | Solución Temporal |
|---------|-----------|------------------|
| Colas | Sin Redis ni workers | Usar `sync` driver (síncrono) |
| Tareas programadas | Sin cron custom | Usar el cron de SiteGround o un trabajo HTTP |
| Escalabilidad | Compartido, recursos limitados | Optimizar consultas, caché con archivos |
| Backup | Automático diario (SiteGround lo provee) | Verificar restauración mensual |

---

## 3. Fase 2 — VPS Propio (Crecimiento)

### 3.1 Especificación Mínima del VPS

| Recurso | Mínimo | Recomendado |
|---------|--------|-------------|
| CPU | 2 cores | 4 cores |
| RAM | 4 GB | 8 GB |
| Disco | 50 GB SSD | 100 GB SSD |
| SO | Ubuntu 24.04 LTS | Ubuntu 24.04 LTS |
| Stack | Nginx + PHP 8.4 + PostgreSQL 16 | Nginx + PHP 8.4 FPM + PostgreSQL 16 + Redis |

### 3.2 Arquitectura en VPS

```
                        ┌──────────────────┐
                        │   Cloudflare     │
                        │  (DNS + CDN)     │
                        └────────┬─────────┘
                                 │
                        ┌────────▼─────────┐
                        │    Nginx         │
                        │  (Reverse Proxy) │
                        └──┬──────────┬────┘
                           │          │
              ┌────────────▼──┐   ┌───▼────────────┐
              │  PHP-FPM      │   │  Archivos       │
              │  (Laravel)    │   │  estáticos      │
              │               │   │  (Frontend)     │
              └───────┬───────┘   └────────────────┘
                      │
         ┌────────────┼────────────┐
         │            │            │
  ┌──────▼─────┐ ┌───▼────┐ ┌────▼─────┐
  │ PostgreSQL │ │ Redis  │ │  Disco   │
  │  (BD)      │ │(Colas) │ │(PDFs,    │
  │            │ │(Cache) │ │ imágenes)│
  └────────────┘ └────────┘ └──────────┘
```

### 3.3 Configuración Nginx

```nginx
server {
    listen 80;
    server_name app.adventur.com admin.adventur.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name app.adventur.com;

    root /var/www/adventur-frontend/dist;
    index index.html;

    ssl_certificate /etc/letsencrypt/live/app.adventur.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.adventur.com/privkey.pem;

    # Frontend SPA
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Backend API
    location /api {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts largos para generación de PDF
        proxy_read_timeout 120s;
    }

    # Archivos estáticos (storage)
    location /storage {
        alias /var/www/adventur-api/storage/app/public;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Seguridad
    location ~ \.env$ {
        deny all;
    }

    # Compresión
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
}
```

### 3.4 Configuración de Colas (Laravel Horizon)

```env
# .env en VPS
QUEUE_CONNECTION=redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
```

```bash
# Supervisor config para mantener colas activas
sudo nano /etc/supervisor/conf.d/adventur-worker.conf
```

```ini
[program:adventur-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/adventur-api/artisan queue:work --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/var/www/adventur-api/storage/logs/worker.log
stopwaitsecs=3600
```

### 3.5 Tareas Programadas (Cron)

```bash
# crontab -e
* * * * * cd /var/www/adventur-api && php artisan schedule:run >> /dev/null 2>&1
```

Tareas programadas en Laravel:

| Tarea | Frecuencia | Descripción |
|-------|-----------|-------------|
| `seguimientos:vencer` | Cada hora | Marcar seguimientos vencidos |
| `cotizaciones:vencer` | Diario 00:00 | Marcar cotizaciones vencidas como Perdido |
| `reservas:finalizar` | Diario 06:00 | Marcar reservas con fecha pasada como Finalizada |
| `backup:diario` | Diario 03:00 | Backup de BD + archivos |
| `auditoria:limpiar` | Mensual | Archivar logs antiguos (opcional) |

### 3.6 Migración de SiteGround a VPS

```
1. Crear VPS con Ubuntu 24.04
2. Instalar stack: Nginx, PHP 8.4, PostgreSQL 16, Redis, Supervisor
3. Clonar repositorio
4. Configurar .env con nuevos valores
5. Copiar backup de BD desde SiteGround:
   pg_dump -U usuario -d adventur_db > backup.sql
   scp backup.sql usuario@vps:/tmp/
   psql -U usuario -d adventur_db < /tmp/backup.sql
6. Copiar archivos storage/ (PDFs, imágenes):
   rsync -avz usuario@siteground:public_html/api/storage/app/ storage/app/
7. Configurar Nginx + SSL (Certbot)
8. Ejecutar migraciones pendientes:
   php artisan migrate
9. Iniciar workers:
   sudo supervisorctl start adventur-worker:*
10. Configurar cron
11. Cambiar DNS de SiteGround al IP del VPS
12. Verificar funcionamiento
```

---

## 4. Fase 3 — Multi-agencia (Futuro)

### 4.1 Cambios Necesarios

| Componente | Cambio |
|-----------|--------|
| Base de Datos | Agregar `id_agencia` en tablas principales |
| Autenticación | Login incluye selección de agencia |
| API | Filtrar por `id_agencia` en todos los endpoints |
| Frontend | Logo y colores configurables por agencia |
| Reportes | Agregar filtro por agencia |
| Usuarios | Asignar usuario a una agencia |

### 4.2 Estructura de Tabla

```sql
ALTER TABLE cotizaciones ADD COLUMN id_agencia INTEGER REFERENCES agencias(id);
ALTER TABLE reservas ADD COLUMN id_agencia INTEGER REFERENCES agencias(id);
ALTER TABLE usuarios ADD COLUMN id_agencia INTEGER REFERENCES agencias(id);

CREATE TABLE agencias (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(255) NOT NULL,
    logo_url TEXT,
    colores JSONB,        -- { "primario": "#...", "secundario": "#..." }
    activo BOOLEAN DEFAULT TRUE,
    fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## 5. Backup y Recuperación

### 5.1 Backup Automático (VPS)

```bash
#!/bin/bash
# /usr/local/bin/backup-adventur.sh

FECHA=$(date +%Y%m%d_%H%M%S)
DIR_BACKUP="/backup/adventur"

mkdir -p $DIR_BACKUP

# Backup de BD
pg_dump -U adventur_user adventur_db | gzip > $DIR_BACKUP/bd_$FECHA.sql.gz

# Backup de archivos
tar -czf $DIR_BACKUP/archivos_$FECHA.tar.gz \
  /var/www/adventur-api/storage/app/

# Eliminar backups mayores a 30 días
find $DIR_BACKUP -name "*.gz" -mtime +30 -delete

# Subir a S3-Compatible (opcional en VPS)
# s3cmd put $DIR_BACKUP/bd_$FECHA.sql.gz s3://adventur-backup/
```

### 5.2 Política de Retención

| Tipo | Retención | Destino |
|------|-----------|---------|
| Backup BD diario | 30 días | Disco local + S3 (opcional) |
| Backup archivos diario | 30 días | Disco local + S3 |
| Backup semanal | 6 meses | S3 |
| Backup mensual | 1 año | S3 |

### 5.3 Procedimiento de Restauración

```bash
# 1. Detener servicios
sudo systemctl stop nginx
sudo systemctl stop php8.4-fpm

# 2. Restaurar BD
gunzip -c /backup/adventur/bd_20260711_030000.sql.gz | psql -U adventur_user adventur_db

# 3. Restaurar archivos
tar -xzf /backup/adventur/archivos_20260711_030000.tar.gz -C /

# 4. Reanudar servicios
sudo systemctl start php8.4-fpm
sudo systemctl start nginx

# 5. Verificar
php artisan adventur:verificar-integridad
```

---

## 6. Monitoreo

### 6.1 Logs

| Log | Ubicación | Retención |
|-----|-----------|-----------|
| Laravel | `storage/logs/laravel.log` | 30 días |
| Workers | `storage/logs/worker.log` | 7 días |
| Nginx | `/var/log/nginx/access.log` | 30 días |
| Nginx Error | `/var/log/nginx/error.log` | 30 días |
| PostgreSQL | `/var/log/postgresql/postgresql.log` | 14 días |

### 6.2 Alertas (VPS)

```bash
# Monitorear disco
df -h / | awk '{print $5}' | tail -1

# Monitorear RAM
free -m | awk 'NR==2{printf "%.2f%%\n", $3*100/$2}'

# Monitorear procesos PHP
ps aux | grep php-fpm | wc -l

# Monitorear workers
sudo supervisorctl status adventur-worker:*
```

---

## 7. CI/CD (GitHub Actions)

### 7.1 Flujo de CI

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  backend-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: adventur_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: 8.4
      - run: composer install
      - run: cp .env.testing .env
      - run: php artisan migrate
      - run: php artisan test

  frontend-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 24
      - run: npm ci
      - run: npm run lint
      - run: npm run build
```

### 7.2 Flujo de CD

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to VPS
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /var/www/adventur-api
            git pull origin main
            composer install --no-dev --optimize-autoloader
            php artisan migrate --force
            php artisan config:cache
            php artisan route:cache
            sudo supervisorctl restart adventur-worker:*
            cd /var/www/adventur-frontend
            git pull origin main
            npm ci
            npm run build
```

---

## 8. Resumen de Costos Estimados

| Recurso | SiteGround (Fase 1) | VPS (Fase 2) |
|---------|:-------------------:|:------------:|
| Hosting | $15/mes (GrowBig) | $20/mes (VPS 4GB) |
| Dominio | $12/año | $12/año |
| SSL | Gratis | Gratis (Let's Encrypt) |
| PostgreSQL | Incluido | Incluido |
| Backup | Incluido (SiteGround) | $5/mes (S3) |
| **Total mes** | **~$16/mes** | **~$26/mes** |
| **Total año** | **~$192/año** | **~$312/año** |
