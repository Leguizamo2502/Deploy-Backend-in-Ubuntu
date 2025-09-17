🚀 Despliegue de miapp en Ubuntu con Nginx y .NET 8 📌 Requisitos previos

Ubuntu 22.04 o superior
Usuario con privilegios sudo
Tener el código fuente de la aplicación en un repositorio Git
Certificado SSL válido o autosignado (opcional si solo usas HTTP) 🛠️ Pasos de instalación y despliegue
1️⃣ Instalar dependencias base

sudo apt-get update -y
sudo apt-get install -y curl wget ca-certificates gnupg lsb-release nginx
Explicación:

apt-get update -y: actualiza índices de paquetes.
apt-get install ... nginx: instala utilidades básicas y Nginx. 2️⃣ Instalar .NET 8 Runtime
# Solo ejecutar aplicaciones ASP.NET Core:
sudo apt-get install -y aspnetcore-runtime-8.0

# Si necesitas compilar en el servidor, usa SDK (opcional):
# sudo apt-get install -y dotnet-sdk-8.0

# Verificar instalación
dotnet --info
Explicación:

aspnetcore-runtime-8.0: runtime para ejecutar apps.
dotnet-sdk-8.0 (comentado): SDK completo (compila y publica). Úsalo solo si vas a construir en el servidor.
dotnet --info: verifica versión y runtimes instalados. 3️⃣ Obtener código (rama sergio) y publicar
cd ~
mkdir -p src && cd src

# Clonar SOLO la rama 'sergio'
git clone --branch sergio --single-branch   https://github.com/Leguizamo2502/Portal-Agro-comercial-del-Huila.git portal-agro
cd portal-agro

# Restaurar y publicar
dotnet restore
dotnet publish -c Release -r linux-x64   --self-contained false   -p:PublishTrimmed=false   -p:PublishSingleFile=false   -o out
Nota: Publicar con -r linux-x64 + --self-contained false es lo correcto si ya tienes el runtime instalado en la máquina. 4️⃣ Preparar directorio de despliegue

sudo mkdir -p /var/www/portal-agro
sudo chown -R $USER:www-data /var/www/portal-agro
sudo chmod -R 750 /var/www/portal-agro

# Limpiar despliegue previo y copiar lo nuevo
sudo rm -rf /var/www/portal-agro/*
sudo cp -r out/* /var/www/portal-agro/

# En producción conviene dejar a www-data como dueño final
sudo chown -R www-data:www-data /var/www/portal-agro
5️⃣ Crear servicio systemd

sudo tee /etc/systemd/system/portal-agro.service >/dev/null <<'EOF'
[Unit]
Description=Portal Agro-comercial del Huila (Kestrel)
After=network.target

[Service]
WorkingDirectory=/var/www/portal-agro
ExecStart=/usr/bin/dotnet /var/www/portal-agro/Web.dll
User=www-data
Group=www-data
Environment=ASPNETCORE_URLS=http://0.0.0.0:5000
EnvironmentFile=-/etc/portal-agro/portal-agro.env
Restart=always
RestartSec=5

NoNewPrivileges=true
ProtectSystem=full
ProtectHome=true

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now portal-agro
sudo systemctl status portal-agro --no-pager
journalctl -u portal-agro -n 200 --no-pager
Nota: Cambia Web.dll por el nombre exacto del DLL publicado. 6️⃣ Configurar NGINX como proxy inverso (HTTP y HTTPS)

sudo tee /etc/nginx/sites-available/portal-agro >/dev/null <<'EOF'
server {
    listen 8082;
    server_name _;

    location / {
        proxy_pass         http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection keep-alive;
        proxy_set_header   Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}

server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /etc/ssl/certs/portal-agro.crt;
    ssl_certificate_key /etc/ssl/private/portal-agro.key;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF

sudo ln -sf /etc/nginx/sites-available/portal-agro /etc/nginx/sites-enabled/portal-agro
sudo rm -f /etc/nginx/sites-enabled/default

sudo nginx -t && sudo systemctl reload nginx
7️⃣ Ciclo de actualización (pull + redeploy rápido)

cd ~/src/portal-agro
git fetch origin
git checkout sergio
git pull --ff-only

dotnet restore
dotnet publish -c Release -r linux-x64 --self-contained false -o out

sudo rm -rf /var/www/portal-agro/*
sudo cp -r out/* /var/www/portal-agro/
sudo chown -R www-data:www-data /var/www/portal-agro

sudo systemctl restart portal-agro
sudo systemctl status portal-agro --no-pager
