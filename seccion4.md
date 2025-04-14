# 4. Implementación de Docker en el Servidor

Docker es una plataforma de contenedores que permite empaquetar aplicaciones y sus dependencias en unidades estandarizadas llamadas contenedores. Estos contenedores son ligeros, portátiles y pueden ejecutarse en cualquier entorno que tenga Docker instalado, lo que facilita enormemente el desarrollo, la implementación y la gestión de aplicaciones.

En esta sección, aprenderemos a instalar Docker y Docker Compose en nuestro servidor Wazuh, configurarlo para que arranque automáticamente con el sistema, y establecer los permisos adecuados para su uso.

## 4.1 Instalación de Docker

### Preparación del Sistema

Antes de instalar Docker, asegurémonos de que el sistema esté actualizado y tenga todas las dependencias necesarias:

```bash
# Actualizar la lista de paquetes
sudo apt update

# Instalar paquetes necesarios para permitir que apt use repositorios sobre HTTPS
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common gnupg
```

### Añadir el Repositorio Oficial de Docker

Vamos a añadir la clave GPG oficial de Docker y su repositorio:

```bash
# Añadir la clave GPG oficial de Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Añadir el repositorio de Docker a las fuentes de APT
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Instalación de Docker Engine

Ahora podemos instalar Docker Engine:

```bash
# Actualizar la lista de paquetes
sudo apt update

# Instalar la última versión de Docker Engine y containerd
sudo apt install -y docker-ce docker-ce-cli containerd.io
```

### Verificación de la Instalación de Docker

Para verificar que Docker se ha instalado correctamente, ejecutemos un contenedor de prueba:

```bash
sudo docker run hello-world
```

Si la instalación fue exitosa, verás un mensaje de confirmación indicando que Docker está funcionando correctamente.

### Comprobar la Versión de Docker

Para verificar la versión de Docker instalada:

```bash
sudo docker --version
```

Deberías ver algo como:

```
Docker version 24.0.5, build 24.0.5-0ubuntu1~22.04.1
```

## 4.2 Instalación de Docker Compose

Docker Compose es una herramienta que permite definir y ejecutar aplicaciones Docker multi-contenedor. Con Compose, utilizas un archivo YAML para configurar los servicios de tu aplicación y luego, con un solo comando, creas e inicias todos los servicios desde tu configuración.

### Instalación de Docker Compose

Existen dos métodos principales para instalar Docker Compose:

#### Método 1: Instalación a través de apt (recomendado)

```bash
# Instalar Docker Compose
sudo apt install -y docker-compose-plugin
```

#### Método 2: Instalación manual

Si prefieres instalar la última versión de Docker Compose manualmente:

```bash
# Descargar la última versión de Docker Compose
COMPOSE_VERSION=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep 'tag_name' | cut -d\" -f4)
sudo curl -L "https://github.com/docker/compose/releases/download/${COMPOSE_VERSION}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Hacer el binario ejecutable
sudo chmod +x /usr/local/bin/docker-compose

# Crear un enlace simbólico
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

### Verificación de la Instalación de Docker Compose

Para verificar que Docker Compose se ha instalado correctamente:

```bash
# Para instalaciones a través de apt
docker compose version

# Para instalaciones manuales
docker-compose --version
```

Deberías ver la versión de Docker Compose instalada.

## 4.3 Configuración de Docker para Arranque Automático

Por defecto, Docker debería configurarse para iniciarse automáticamente con el sistema. Podemos verificarlo y asegurarnos de que sea así:

```bash
# Verificar el estado actual de Docker
sudo systemctl status docker
```

Si Docker no está configurado para iniciarse automáticamente, podemos habilitarlo:

```bash
# Habilitar Docker para que se inicie con el sistema
sudo systemctl enable docker

# Iniciar Docker si no está en ejecución
sudo systemctl start docker
```

### Configuración de Parámetros de Docker

Para personalizar la configuración de Docker, podemos editar el archivo de configuración del daemon:

```bash
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json
```

Añade la siguiente configuración básica:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-address-pools": [
    {
      "base": "172.17.0.0/16",
      "size": 24
    }
  ]
}
```

Esta configuración:
- Limita el tamaño de los archivos de log a 10MB y mantiene un máximo de 3 archivos
- Configura el rango de direcciones IP por defecto para las redes de Docker

Guarda el archivo y reinicia Docker para aplicar los cambios:

```bash
sudo systemctl restart docker
```

## 4.4 Ajustes de Red para Docker

### Creación de una Red Personalizada

Es una buena práctica crear redes personalizadas para tus contenedores en lugar de utilizar la red por defecto de Docker:

```bash
# Crear una red bridge personalizada
sudo docker network create --driver bridge wazuh-net
```

### Verificación de Redes Docker

Para ver las redes disponibles en Docker:

```bash
sudo docker network ls
```

Deberías ver la red `wazuh-net` que acabamos de crear, junto con las redes predeterminadas de Docker.

## 4.5 Gestión de Permisos y Configuración de Usuarios para Docker

Por defecto, el comando `docker` requiere privilegios de root, lo que significa que debes usar `sudo` cada vez que ejecutas un comando de Docker. Para evitar esto, podemos añadir nuestro usuario al grupo `docker`.

### Creación del Grupo Docker

El grupo `docker` debería haberse creado durante la instalación, pero podemos verificarlo:

```bash
# Verificar si el grupo docker existe
getent group docker
```

Si el grupo no existe, podemos crearlo:

```bash
sudo groupadd docker
```

### Añadir Usuario al Grupo Docker

Ahora, añadamos nuestro usuario al grupo `docker`:

```bash
# Añadir el usuario actual al grupo docker
sudo usermod -aG docker $USER
```

Para aplicar los cambios de grupo sin cerrar sesión:

```bash
newgrp docker
```

### Verificación de Permisos

Para verificar que los permisos se han configurado correctamente:

```bash
# Ejecutar un contenedor sin sudo
docker run hello-world
```

Si el comando se ejecuta sin errores, significa que los permisos están configurados correctamente.

### Consideraciones de Seguridad

Añadir usuarios al grupo `docker` les otorga efectivamente privilegios de root en el sistema, ya que pueden montar volúmenes y acceder a recursos del sistema a través de contenedores. Por lo tanto, solo debes añadir usuarios de confianza al grupo `docker`.

Para entornos de producción, considera implementar medidas de seguridad adicionales:

1. **Auditoría de Acceso**: Configura la auditoría para monitorear las acciones realizadas con Docker:

```bash
sudo apt install -y auditd
sudo nano /etc/audit/rules.d/docker.rules
```

Añade las siguientes reglas:

```
-w /usr/bin/docker -p wa -k docker
-w /var/lib/docker -p wa -k docker
-w /etc/docker -p wa -k docker
-w /usr/lib/systemd/system/docker.service -p wa -k docker
-w /usr/lib/systemd/system/docker.socket -p wa -k docker
-w /etc/default/docker -p wa -k docker
-w /etc/docker/daemon.json -p wa -k docker
```

Reinicia el servicio de auditoría:

```bash
sudo systemctl restart auditd
```

2. **Limitar Recursos**: Utiliza cgroups para limitar los recursos que los contenedores pueden utilizar:

```bash
# Ejemplo: limitar la memoria y CPU de un contenedor
docker run --memory=512m --cpus=0.5 -d nginx
```

3. **Escaneo de Vulnerabilidades**: Implementa herramientas de escaneo de vulnerabilidades para contenedores:

```bash
# Instalar Trivy
sudo apt install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install -y trivy

# Escanear una imagen
trivy image nginx:latest
```

## 4.6 Verificación de la Instalación de Docker

Para asegurarnos de que Docker está correctamente instalado y configurado, realizaremos algunas verificaciones adicionales:

### Verificación del Servicio Docker

```bash
# Verificar el estado del servicio Docker
sudo systemctl status docker
```

El servicio debería estar activo (running) y habilitado.

### Verificación de la Configuración del Daemon

```bash
# Verificar la configuración del daemon de Docker
docker info
```

Este comando mostrará información detallada sobre la instalación de Docker, incluyendo la versión, el número de contenedores e imágenes, la configuración de almacenamiento, etc.

### Prueba de Funcionalidad Básica

```bash
# Ejecutar un contenedor de prueba
docker run --rm alpine echo "Docker está funcionando correctamente"
```

Si ves el mensaje "Docker está funcionando correctamente", significa que Docker puede descargar imágenes y ejecutar contenedores sin problemas.

### Verificación de Docker Compose

```bash
# Crear un archivo docker-compose.yml de prueba
cat > docker-compose-test.yml << EOF
version: '3'
services:
  hello-world:
    image: hello-world
EOF

# Ejecutar el archivo de prueba
docker compose -f docker-compose-test.yml up

# Limpiar después de la prueba
docker compose -f docker-compose-test.yml down
rm docker-compose-test.yml
```

Si Docker Compose funciona correctamente, verás el mensaje de salida del contenedor hello-world.

## 4.7 Solución de Problemas Comunes

### Problema: Error "Permission denied"

**Síntoma**: Al ejecutar comandos Docker sin `sudo`, aparece un error de permisos.

**Solución**:
1. Asegúrate de haber añadido tu usuario al grupo `docker`:
   ```bash
   sudo usermod -aG docker $USER
   ```
2. Cierra sesión y vuelve a iniciarla, o ejecuta:
   ```bash
   newgrp docker
   ```

### Problema: Docker no Inicia Automáticamente

**Síntoma**: Docker no se inicia automáticamente después de reiniciar el sistema.

**Solución**:
1. Habilita el servicio Docker:
   ```bash
   sudo systemctl enable docker
   ```
2. Verifica si hay errores en los logs:
   ```bash
   sudo journalctl -u docker
   ```

### Problema: Conflictos de Red

**Síntoma**: Errores al crear redes o conectar contenedores a redes.

**Solución**:
1. Verifica las redes existentes:
   ```bash
   docker network ls
   ```
2. Elimina y recrea la red si es necesario:
   ```bash
   docker network rm wazuh-net
   docker network create --driver bridge wazuh-net
   ```
3. Verifica si hay conflictos con otras redes en el sistema:
   ```bash
   ip addr show
   ```

### Problema: Espacio en Disco Insuficiente

**Síntoma**: Errores relacionados con espacio en disco insuficiente.

**Solución**:
1. Limpia recursos no utilizados:
   ```bash
   docker system prune -a
   ```
2. Verifica el espacio utilizado por Docker:
   ```bash
   docker system df
   ```
3. Considera mover el directorio de datos de Docker a un volumen con más espacio:
   ```bash
   # Detener Docker
   sudo systemctl stop docker
   
   # Mover datos existentes
   sudo rsync -aqxP /var/lib/docker/ /nuevo/path/docker
   
   # Configurar nueva ubicación
   sudo nano /etc/docker/daemon.json
   # Añadir: {"data-root": "/nuevo/path/docker"}
   
   # Reiniciar Docker
   sudo systemctl start docker
   ```

Con Docker y Docker Compose correctamente instalados y configurados, ahora tenemos la base necesaria para trabajar con contenedores. En la siguiente sección, crearemos y configuraremos contenedores de prueba que utilizaremos para demostrar el monitoreo con Wazuh.
