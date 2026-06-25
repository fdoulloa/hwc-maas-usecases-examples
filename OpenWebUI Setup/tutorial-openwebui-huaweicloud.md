# Tutorial: Despliegue de Open WebUI en Huawei Cloud ECS con Docker

## Descripción general

Este tutorial describe el proceso completo para aprovisionar una instancia ECS (Elastic Cloud Server) en Huawei Cloud, instalar Docker y desplegar Open WebUI como contenedor. Al finalizar, tendrás una interfaz web funcional para interactuar con modelos de lenguaje (LLM).

---

## Requisitos previos

- Cuenta activa en Huawei Cloud con permisos para crear recursos ECS, VPC y Security Groups
- Acceso a la consola web de Huawei Cloud (https://console.huaweicloud.com)
- Par de claves SSH creado previamente (o se puede crear durante el proceso)
- Conocimientos básicos de Linux y línea de comandos

---

## Arquitectura del despliegue

```
Internet
   │
   ▼
[EIP - Elastic IP]
   │
   ▼
[ECS - Ubuntu 22.04]
   ├── Docker Engine
   └── Open WebUI (contenedor, puerto 3000)
        └── [Opcional] Ollama (contenedor, puerto 11434)
```

---

## Paso 1: Crear una VPC y Subnet

1. En la consola de Huawei Cloud, ir a **Virtual Private Cloud (VPC)**.
2. Hacer clic en **Create VPC**.
3. Completar los campos:
   - **Name:** `vpc-openwebui`
   - **IPv4 CIDR block:** `192.168.0.0/16`
   - **Subnet name:** `subnet-openwebui`
   - **Subnet CIDR:** `192.168.1.0/24`
   - **Availability Zone:** seleccionar la AZ de preferencia
4. Hacer clic en **Create Now**.

---

## Paso 2: Crear un Security Group

1. En la consola, ir a **VPC → Security Groups**.
2. Hacer clic en **Create Security Group**.
3. Asignar nombre: `sg-openwebui`.
4. Agregar las siguientes reglas de **Inbound**:

| Prioridad | Protocolo | Puerto | Fuente | Descripción |
|-----------|-----------|--------|--------|-------------|
| 100 | TCP | 22 | Tu IP / 0.0.0.0/0 | Acceso SSH |
| 100 | TCP | 3000 | 0.0.0.0/0 | Open WebUI |
| 100 | TCP | 11434 | 192.168.1.0/24 | Ollama (solo interno, opcional) |

> **Nota de seguridad:** Limitar el puerto 22 únicamente a tu IP pública es una práctica recomendada.

5. Dejar las reglas **Outbound** en `Allow All` (configuración por defecto).
6. Hacer clic en **OK**.

---

## Paso 3: Crear el Key Pair (par de claves SSH)

> Omitir este paso si ya cuentas con un key pair registrado en Huawei Cloud.

1. Ir a **Elastic Cloud Server → Key Pairs**.
2. Hacer clic en **Create Key Pair**.
3. Asignar nombre: `kp-openwebui`.
4. Hacer clic en **OK**. El archivo `.pem` se descargará automáticamente.
5. Guardar el archivo en un lugar seguro y ajustar permisos:

```bash
chmod 400 kp-openwebui.pem
```

---

## Paso 4: Crear la instancia ECS

### 4.1 Acceder al asistente de creación

1. En la consola, ir a **Elastic Cloud Server**.
2. Hacer clic en **Buy ECS**.

### 4.2 Configuración básica

| Parámetro | Valor recomendado |
|-----------|-------------------|
| **Billing Mode** | Pay-per-use (o Yearly/Monthly según necesidad) |
| **Region** | La región más cercana a tus usuarios |
| **AZ** | AZ preferida dentro de la región |
| **Flavor** | `c7.xlarge.2` (4 vCPU, 8 GB RAM) mínimo recomendado |
| **Image** | Ubuntu Server 22.04 LTS (64bit) |
| **System Disk** | SSD General Purpose, 40 GB mínimo |

> **Para uso con Ollama (LLM local):** Se recomienda mínimo `c7.2xlarge.4` (8 vCPU, 16 GB RAM) o instancias GPU si se trabaja con modelos grandes.

### 4.3 Configuración de red

| Parámetro | Valor |
|-----------|-------|
| **VPC** | `vpc-openwebui` |
| **Subnet** | `subnet-openwebui` |
| **Security Group** | `sg-openwebui` |
| **EIP** | Auto Assign (seleccionar ancho de banda según necesidad) |

### 4.4 Credenciales de acceso

- **Login Mode:** Key pair
- **Key Pair:** `kp-openwebui`

### 4.5 Nombre y confirmación

- **ECS Name:** `ecs-openwebui`
- Revisar el resumen y hacer clic en **Submit**.

---

## Paso 5: Conectarse a la instancia por SSH

Una vez que la ECS esté en estado **Running** (1-3 minutos):

```bash
ssh -i kp-openwebui.pem ubuntu@<EIP_de_tu_ECS>
```

Verificar conectividad:

```bash
uname -a
# Debería mostrar algo similar a:
# Linux ecs-openwebui 5.15.0-... x86_64 GNU/Linux
```

---

## Paso 6: Preparar el sistema operativo

```bash
# Actualizar paquetes del sistema
sudo apt update && sudo apt upgrade -y

# Instalar dependencias base
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    wget \
    git \
    htop
```

---

## Paso 7: Instalar Docker Engine

### 7.1 Agregar el repositorio oficial de Docker

```bash
# Crear directorio para keyrings
sudo install -m 0755 -d /etc/apt/keyrings

# Descargar y agregar la clave GPG de Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Agregar el repositorio de Docker a las fuentes de apt
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 7.2 Instalar Docker

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 7.3 Habilitar Docker y agregar usuario al grupo

```bash
# Iniciar y habilitar Docker al arranque
sudo systemctl enable docker
sudo systemctl start docker

# Agregar el usuario actual al grupo docker (evita usar sudo en cada comando)
sudo usermod -aG docker $USER

# Aplicar el cambio de grupo en la sesión actual
newgrp docker
```

### 7.4 Verificar instalación

```bash
docker --version
# Docker version 24.x.x, build ...

docker run hello-world
# Debe imprimir "Hello from Docker!"
```

---

## Paso 8: Desplegar Open WebUI

Existen dos modos de despliegue según si se usará un proveedor de LLM externo (como Huawei MaaS/ModelArts) o Ollama local.

---

### Opción A: Open WebUI con proveedor LLM externo (API compatible con OpenAI)

Este modo conecta Open WebUI a un endpoint externo (como Huawei ModelArts o cualquier API compatible con OpenAI).

```bash
docker run -d \
  --name open-webui \
  --restart always \
  -p 3000:8080 \
  -e OPENAI_API_BASE_URL="https://<tu-endpoint-modelarts>/v1" \
  -e OPENAI_API_KEY="<tu-api-key>" \
  -v open-webui:/app/backend/data \
  ghcr.io/open-webui/open-webui:main
```

> Reemplazar `<tu-endpoint-modelarts>` y `<tu-api-key>` con los valores de tu servicio LLM.

---

### Opción B: Open WebUI + Ollama (LLM local en la misma ECS)

#### 8.1 Instalar Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh

# Verificar que Ollama esté corriendo
sudo systemctl status ollama
```

#### 8.2 Descargar un modelo de ejemplo

```bash
# Modelo liviano para pruebas (3.8 GB)
ollama pull llama3.2:3b

# Listar modelos descargados
ollama list
```

#### 8.3 Levantar Open WebUI conectado a Ollama local

```bash
docker run -d \
  --name open-webui \
  --restart always \
  --network=host \
  -e OLLAMA_BASE_URL=http://127.0.0.1:11434 \
  -v open-webui:/app/backend/data \
  ghcr.io/open-webui/open-webui:main
```

> Se usa `--network=host` para que el contenedor pueda alcanzar Ollama en `localhost`.

---

## Paso 9: Verificar el despliegue

### 9.1 Comprobar que el contenedor está corriendo

```bash
docker ps

# Salida esperada:
# CONTAINER ID   IMAGE                                COMMAND        STATUS         PORTS
# abc123def456   ghcr.io/open-webui/open-webui:main  ...            Up X minutes   0.0.0.0:3000->8080/tcp
```

### 9.2 Revisar logs del contenedor

```bash
docker logs -f open-webui
```

Esperar hasta ver líneas similares a:
```
INFO:     Uvicorn running on http://0.0.0.0:8080 (Press CTRL+C to quit)
```

### 9.3 Acceder a la interfaz web

Abrir en el navegador:

```
http://<EIP_de_tu_ECS>:3000
```

En el primer acceso, Open WebUI solicitará crear una cuenta de administrador local.

---

## Paso 10: Configuración inicial de Open WebUI

1. Completar el formulario de registro con **nombre**, **email** y **contraseña** para la cuenta admin.
2. Hacer clic en **Create Admin Account**.
3. En la pantalla principal, verificar que los modelos aparezcan en el selector superior izquierdo.
4. Para agregar conexiones a APIs externas adicionales:
   - Ir a **Settings → Connections**
   - Agregar el endpoint y API key del proveedor LLM deseado

---

## Paso 11 (Opcional): Usar Docker Compose

Para una gestión más ordenada, especialmente con múltiples servicios, se puede usar Docker Compose.

### Crear el archivo `docker-compose.yml`

```bash
mkdir ~/openwebui && cd ~/openwebui
nano docker-compose.yml
```

Pegar el siguiente contenido (modo con Ollama):

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: always
    volumes:
      - ollama_data:/root/.ollama
    ports:
      - "11434:11434"

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: always
    depends_on:
      - ollama
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    volumes:
      - open-webui_data:/app/backend/data

volumes:
  ollama_data:
  open-webui_data:
```

### Levantar los servicios

```bash
docker compose up -d

# Ver estado
docker compose ps

# Ver logs
docker compose logs -f open-webui
```

---

## Paso 12 (Opcional): Configurar HTTPS con Nginx

Para entornos productivos se recomienda exponer Open WebUI a través de HTTPS.

### 12.1 Instalar Nginx y Certbot

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

### 12.2 Crear la configuración de Nginx

```bash
sudo nano /etc/nginx/sites-available/openwebui
```

```nginx
server {
    listen 80;
    server_name <tu-dominio.com>;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 300s;
    }
}
```

### 12.3 Activar el sitio y obtener certificado SSL

```bash
sudo ln -s /etc/nginx/sites-available/openwebui /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# Obtener certificado SSL (requiere dominio apuntando a la EIP)
sudo certbot --nginx -d <tu-dominio.com>
```

> Si se usa SSL, agregar el puerto 443 en el Security Group y reemplazar el acceso por `https://<tu-dominio.com>`.

---

## Comandos útiles de administración

```bash
# Detener Open WebUI
docker stop open-webui

# Iniciar Open WebUI
docker start open-webui

# Reiniciar Open WebUI
docker restart open-webui

# Ver uso de recursos del contenedor
docker stats open-webui

# Actualizar a la última versión de Open WebUI
docker pull ghcr.io/open-webui/open-webui:main
docker stop open-webui && docker rm open-webui
# Volver a ejecutar el comando docker run del Paso 8

# Ver volúmenes persistentes
docker volume ls

# Ver logs en tiempo real
docker logs -f open-webui --tail 100
```

---

## Solución de problemas comunes

| Problema | Causa probable | Solución |
|----------|----------------|----------|
| No se puede acceder al puerto 3000 | Regla faltante en Security Group | Verificar inbound rule TCP 3000 en `sg-openwebui` |
| Contenedor sale inmediatamente | Error en variables de entorno | Revisar `docker logs open-webui` |
| Open WebUI no ve modelos Ollama | Error de red entre contenedores | Verificar que se use `--network=host` o el nombre de servicio en Compose |
| SSH rechazado | Permisos del `.pem` incorrecto | Ejecutar `chmod 400 kp-openwebui.pem` |
| Disco lleno | Modelos LLM ocupan mucho espacio | Ampliar disco de datos ECS o usar EVS adicional |
| Imagen no descarga (`ghcr.io`) | Restricción de red saliente | Verificar reglas outbound del Security Group |

---

## Resumen de puertos y servicios

| Servicio | Puerto interno | Puerto expuesto | Descripción |
|----------|---------------|-----------------|-------------|
| Open WebUI | 8080 | 3000 | Interfaz web |
| Ollama | 11434 | 11434 (solo interno) | Motor LLM local |
| Nginx (HTTPS) | — | 443 | Proxy reverso SSL |
| SSH | 22 | 22 | Administración |

---

*Versión del tutorial: 1.0 — Compatible con Open WebUI versión main y Docker Engine 24+*
