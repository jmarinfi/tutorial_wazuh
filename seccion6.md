# 6. Instalación y Configuración del Agente Wazuh en Contenedores

Una vez que hemos creado y configurado nuestros contenedores de prueba, el siguiente paso es instalar y configurar el agente Wazuh en ellos. Esto nos permitirá monitorear la actividad dentro de los contenedores y detectar posibles amenazas o vulnerabilidades.

En esta sección, exploraremos diferentes métodos para instalar el agente Wazuh en contenedores Docker, configurarlo para conectarse con el servidor Wazuh, registrarlo correctamente y verificar su funcionamiento.

## 6.1 Métodos de Instalación del Agente Wazuh en Contenedores

Existen varios enfoques para instalar el agente Wazuh en contenedores Docker. Vamos a explorar los más comunes y efectivos.

### Instalación Directa en Contenedores Existentes

El método más sencillo es instalar el agente directamente en contenedores que ya están en ejecución. Este enfoque es similar a la instalación del agente en un sistema operativo tradicional.

#### Instalación en Contenedor Ubuntu

Vamos a instalar el agente Wazuh en nuestro contenedor Ubuntu de prueba:

```bash
# Acceder al contenedor
docker exec -it ubuntu-test bash

# Dentro del contenedor, actualizar e instalar dependencias
apt update
apt install -y curl apt-transport-https gnupg

# Añadir la clave GPG de Wazuh
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | apt-key add -

# Añadir el repositorio de Wazuh
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | tee /etc/apt/sources.list.d/wazuh.list

# Actualizar e instalar el agente Wazuh
apt update
apt install -y wazuh-agent

# Salir del contenedor
exit
```

#### Instalación en Contenedor Debian

De manera similar, instalamos el agente en el contenedor Debian:

```bash
# Acceder al contenedor
docker exec -it debian-test bash

# Dentro del contenedor, actualizar e instalar dependencias
apt update
apt install -y curl apt-transport-https gnupg

# Añadir la clave GPG de Wazuh
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | apt-key add -

# Añadir el repositorio de Wazuh
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | tee /etc/apt/sources.list.d/wazuh.list

# Actualizar e instalar el agente Wazuh
apt update
apt install -y wazuh-agent

# Salir del contenedor
exit
```

### Creación de Imágenes Personalizadas con el Agente Preinstalado

Otra aproximación es crear imágenes Docker personalizadas que ya incluyan el agente Wazuh. Esto es especialmente útil cuando necesitas desplegar múltiples contenedores con el agente instalado.

#### Creación de un Dockerfile para Ubuntu con Agente Wazuh

Vamos a crear un Dockerfile que extienda la imagen de Ubuntu e incluya el agente Wazuh:

```bash
# Crear un directorio para el Dockerfile
mkdir -p ~/wazuh-docker/ubuntu-agent
cd ~/wazuh-docker/ubuntu-agent

# Crear el Dockerfile
cat > Dockerfile << EOF
FROM ubuntu:22.04

# Evitar interacciones durante la instalación de paquetes
ENV DEBIAN_FRONTEND=noninteractive

# Instalar dependencias
RUN apt-get update && \
    apt-get install -y curl apt-transport-https gnupg systemd && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Añadir el repositorio de Wazuh
RUN curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | apt-key add - && \
    echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | tee /etc/apt/sources.list.d/wazuh.list && \
    apt-get update

# Instalar el agente Wazuh
RUN apt-get install -y wazuh-agent

# Mantener el contenedor en ejecución
CMD ["/bin/bash", "-c", "sleep infinity"]
EOF

# Construir la imagen
docker build -t ubuntu-wazuh-agent:latest .
```

#### Creación de un Dockerfile para Debian con Agente Wazuh

De manera similar, creamos un Dockerfile para Debian:

```bash
# Crear un directorio para el Dockerfile
mkdir -p ~/wazuh-docker/debian-agent
cd ~/wazuh-docker/debian-agent

# Crear el Dockerfile
cat > Dockerfile << EOF
FROM debian:11

# Evitar interacciones durante la instalación de paquetes
ENV DEBIAN_FRONTEND=noninteractive

# Instalar dependencias
RUN apt-get update && \
    apt-get install -y curl apt-transport-https gnupg systemd && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Añadir el repositorio de Wazuh
RUN curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | apt-key add - && \
    echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | tee /etc/apt/sources.list.d/wazuh.list && \
    apt-get update

# Instalar el agente Wazuh
RUN apt-get install -y wazuh-agent

# Mantener el contenedor en ejecución
CMD ["/bin/bash", "-c", "sleep infinity"]
EOF

# Construir la imagen
docker build -t debian-wazuh-agent:latest .
```

#### Ejecución de Contenedores con las Imágenes Personalizadas

Ahora podemos ejecutar contenedores utilizando nuestras imágenes personalizadas:

```bash
# Ejecutar un contenedor Ubuntu con el agente Wazuh preinstalado
docker run -d \
  --name ubuntu-wazuh \
  --hostname ubuntu-wazuh \
  --network wazuh-net \
  -v ubuntu-wazuh-data:/var/ossec/etc \
  --restart unless-stopped \
  ubuntu-wazuh-agent:latest

# Ejecutar un contenedor Debian con el agente Wazuh preinstalado
docker run -d \
  --name debian-wazuh \
  --hostname debian-wazuh \
  --network wazuh-net \
  -v debian-wazuh-data:/var/ossec/etc \
  --restart unless-stopped \
  debian-wazuh-agent:latest
```

### Uso de Docker Compose para Desplegar Contenedores con Agente Wazuh

También podemos utilizar Docker Compose para definir y desplegar contenedores con el agente Wazuh:

```bash
# Crear un directorio para el proyecto
mkdir -p ~/wazuh-docker-agents
cd ~/wazuh-docker-agents

# Crear el archivo docker-compose.yml
cat > docker-compose.yml << EOF
version: '3'

services:
  ubuntu-wazuh:
    build:
      context: ./ubuntu-agent
    container_name: ubuntu-wazuh
    hostname: ubuntu-wazuh
    restart: unless-stopped
    networks:
      - wazuh-net
    volumes:
      - ubuntu-wazuh-data:/var/ossec/etc

  debian-wazuh:
    build:
      context: ./debian-agent
    container_name: debian-wazuh
    hostname: debian-wazuh
    restart: unless-stopped
    networks:
      - wazuh-net
    volumes:
      - debian-wazuh-data:/var/ossec/etc

networks:
  wazuh-net:
    external: true

volumes:
  ubuntu-wazuh-data:
  debian-wazuh-data:
EOF

# Crear los directorios para los Dockerfiles
mkdir -p ubuntu-agent debian-agent

# Copiar los Dockerfiles creados anteriormente
cp ~/wazuh-docker/ubuntu-agent/Dockerfile ubuntu-agent/
cp ~/wazuh-docker/debian-agent/Dockerfile debian-agent/

# Desplegar los contenedores
docker compose up -d
```

## 6.2 Configuración del Agente Wazuh en Contenedores

Una vez instalado el agente Wazuh en los contenedores, necesitamos configurarlo para que se conecte con el servidor Wazuh.

### Configuración Básica del Agente

La configuración principal del agente Wazuh se encuentra en el archivo `/var/ossec/etc/ossec.conf`. Vamos a modificar este archivo en nuestros contenedores:

```bash
# Acceder al contenedor Ubuntu
docker exec -it ubuntu-test bash

# Editar el archivo de configuración
nano /var/ossec/etc/ossec.conf
```

Busca la sección `<client>` y modifícala para que apunte a tu servidor Wazuh:

```xml
<client>
  <server>
    <address>IP_DEL_SERVIDOR_WAZUH</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
  <config-profile>ubuntu, ubuntu22</config-profile>
  <notify_time>10</notify_time>
  <time-reconnect>60</time-reconnect>
  <auto_restart>yes</auto_restart>
  <crypto_method>aes</crypto_method>
</client>
```

Reemplaza `IP_DEL_SERVIDOR_WAZUH` con la dirección IP de tu servidor Wazuh.

### Configuración para Monitoreo de Contenedores

Para mejorar el monitoreo específico de contenedores, podemos añadir configuraciones adicionales:

```xml
<!-- Monitoreo de archivos específicos de Docker -->
<localfile>
  <log_format>syslog</log_format>
  <location>/var/lib/docker/containers/*/*.log</location>
</localfile>

<!-- Monitoreo de eventos de Docker -->
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/docker.log</location>
</localfile>

<!-- Monitoreo de cambios en archivos de configuración -->
<syscheck>
  <directories check_all="yes">/etc/docker</directories>
  <directories check_all="yes">/var/lib/docker/volumes</directories>
</syscheck>
```

### Configuración Mediante Variables de Entorno

También podemos configurar el agente Wazuh utilizando variables de entorno al iniciar el contenedor:

```bash
docker run -d \
  --name ubuntu-wazuh-env \
  --hostname ubuntu-wazuh-env \
  --network wazuh-net \
  -e WAZUH_MANAGER=IP_DEL_SERVIDOR_WAZUH \
  -e WAZUH_AGENT_NAME=ubuntu-docker-env \
  -e WAZUH_AGENT_GROUP=docker,ubuntu \
  ubuntu-wazuh-agent:latest
```

Para que esto funcione, necesitamos modificar nuestro Dockerfile para que procese estas variables de entorno:

```bash
# Modificar el Dockerfile
cat > ~/wazuh-docker/ubuntu-agent/Dockerfile << EOF
FROM ubuntu:22.04

# Evitar interacciones durante la instalación de paquetes
ENV DEBIAN_FRONTEND=noninteractive

# Instalar dependencias
RUN apt-get update && \
    apt-get install -y curl apt-transport-https gnupg systemd && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Añadir el repositorio de Wazuh
RUN curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | apt-key add - && \
    echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | tee /etc/apt/sources.list.d/wazuh.list && \
    apt-get update

# Instalar el agente Wazuh
RUN apt-get install -y wazuh-agent

# Script de inicialización
COPY init.sh /
RUN chmod +x /init.sh

# Ejecutar script de inicialización
ENTRYPOINT ["/init.sh"]
EOF

# Crear el script de inicialización
cat > ~/wazuh-docker/ubuntu-agent/init.sh << EOF
#!/bin/bash

# Configurar el agente Wazuh con variables de entorno
if [ ! -z \$WAZUH_MANAGER ]; then
    sed -i "s/<address>.*<\/address>/<address>\${WAZUH_MANAGER}<\/address>/" /var/ossec/etc/ossec.conf
fi

if [ ! -z \$WAZUH_AGENT_NAME ]; then
    sed -i "s/<client_name>.*<\/client_name>/<client_name>\${WAZUH_AGENT_NAME}<\/client_name>/" /var/ossec/etc/ossec.conf
fi

if [ ! -z \$WAZUH_AGENT_GROUP ]; then
    echo "\${WAZUH_AGENT_GROUP}" > /var/ossec/etc/shared/agent.conf
fi

# Iniciar el agente Wazuh
/var/ossec/bin/wazuh-control start

# Mantener el contenedor en ejecución
exec tail -f /var/ossec/logs/ossec.log
EOF

# Reconstruir la imagen
cd ~/wazuh-docker/ubuntu-agent
docker build -t ubuntu-wazuh-agent:latest .
```

## 6.3 Registro del Agente en el Servidor Wazuh

Para que el agente Wazuh pueda comunicarse con el servidor, debe estar registrado. Existen varias formas de realizar este registro.

### Registro Manual

El método más básico es el registro manual:

```bash
# En el servidor Wazuh, generar una clave para el agente
sudo /var/ossec/bin/manage_agents -a -n "ubuntu-docker" -i any

# La salida mostrará una clave larga, cópiala

# En el contenedor con el agente Wazuh
docker exec -it ubuntu-test bash

# Importar la clave
/var/ossec/bin/manage_agents -i CLAVE_COPIADA

# Reiniciar el agente
/var/ossec/bin/wazuh-control restart

# Salir del contenedor
exit
```

### Registro Automático

Para entornos con muchos agentes, es más eficiente utilizar el registro automático:

1. Primero, habilitamos el registro automático en el servidor Wazuh:

```bash
# En el servidor Wazuh
sudo nano /var/ossec/etc/ossec.conf
```

Busca o añade la sección `<auth>`:

```xml
<auth>
  <disabled>no</disabled>
  <port>1515</port>
  <use_source_ip>no</use_source_ip>
  <force_insert>yes</force_insert>
  <force_time>0</force_time>
  <purge>yes</purge>
  <use_password>no</use_password>
  <limit_maxagents>yes</limit_maxagents>
  <ciphers>HIGH:!ADH:!EXP:!MD5:!RC4:!3DES:!CAMELLIA:@STRENGTH</ciphers>
  <!-- <ssl_agent_ca></ssl_agent_ca> -->
  <ssl_verify_host>no</ssl_verify_host>
  <ssl_manager_cert>/var/ossec/etc/sslmanager.cert</ssl_manager_cert>
  <ssl_manager_key>/var/ossec/etc/sslmanager.key</ssl_manager_key>
  <ssl_auto_negotiate>no</ssl_auto_negotiate>
</auth>
```

Reinicia el servidor Wazuh:

```bash
sudo systemctl restart wazuh-manager
```

2. Luego, configuramos el agente para el registro automático:

```bash
# Acceder al contenedor
docker exec -it ubuntu-test bash

# Editar el archivo de configuración
nano /var/ossec/etc/ossec.conf
```

Asegúrate de que la sección `<client>` incluya la dirección del servidor:

```xml
<client>
  <server>
    <address>IP_DEL_SERVIDOR_WAZUH</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

3. Ejecuta el registro automático:

```bash
/var/ossec/bin/agent-auth -m IP_DEL_SERVIDOR_WAZUH -p 1515
```

4. Reinicia el agente:

```bash
/var/ossec/bin/wazuh-control restart
```

### Registro Mediante API REST

También podemos utilizar la API REST de Wazuh para registrar agentes:

1. Primero, obtenemos un token de autenticación:

```bash
# En el host, no en el contenedor
curl -k -u admin:admin -X POST "https://IP_DEL_SERVIDOR_WAZUH:55000/security/user/authenticate?raw=true"
```

Esto devolverá un token JWT que utilizaremos en las siguientes peticiones.

2. Registramos el agente:

```bash
curl -k -X POST "https://IP_DEL_SERVIDOR_WAZUH:55000/agents" \
  -H "Authorization: Bearer TOKEN_JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "ubuntu-docker-api",
    "ip": "any"
  }'
```

La respuesta incluirá el ID del agente y la clave.

3. Importamos la clave en el agente:

```bash
# Acceder al contenedor
docker exec -it ubuntu-test bash

# Importar la clave
echo "CLAVE_DEL_AGENTE" | /var/ossec/bin/manage_agents -i

# Reiniciar el agente
/var/ossec/bin/wazuh-control restart
```

## 6.4 Verificación de la Comunicación Agente-Servidor

Una vez que el agente está instalado, configurado y registrado, debemos verificar que la comunicación con el servidor Wazuh funciona correctamente.

### Verificación en el Agente

```bash
# Acceder al contenedor
docker exec -it ubuntu-test bash

# Verificar el estado del agente
/var/ossec/bin/wazuh-control status

# Verificar los logs del agente
tail -f /var/ossec/logs/ossec.log
```

Deberías ver mensajes indicando que el agente se ha conectado correctamente al servidor.

### Verificación en el Servidor

```bash
# En el servidor Wazuh
sudo /var/ossec/bin/agent_control -l
```

Este comando mostrará una lista de todos los agentes conectados. Deberías ver tu agente Docker en la lista con estado "Active".

### Verificación a través del Dashboard

También puedes verificar la conexión a través del Wazuh dashboard:

1. Accede al dashboard en `https://IP_DEL_SERVIDOR_WAZUH`
2. Ve a la sección "Agents"
3. Deberías ver tu agente Docker en la lista con estado "Active"

## 6.5 Opciones Avanzadas de Configuración del Agente

El agente Wazuh ofrece numerosas opciones de configuración avanzadas que pueden ser especialmente útiles para el monitoreo de contenedores.

### Monitoreo de Archivos Específicos

Para monitorear archivos específicos dentro del contenedor:

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/app.log</location>
</localfile>
```

### Detección de Cambios en Contenedores

Para detectar cambios en archivos críticos:

```xml
<syscheck>
  <directories check_all="yes">/app</directories>
  <directories check_all="yes">/etc</directories>
  <directories check_all="yes">/var/www/html</directories>
</syscheck>
```

### Alertas Personalizadas

Podemos configurar alertas personalizadas para eventos específicos:

```xml
<command>
  <name>custom-alert</name>
  <executable>custom-script.sh</executable>
  <expect>srcip</expect>
  <timeout_allowed>yes</timeout_allowed>
</command>

<active-response>
  <command>custom-alert</command>
  <location>local</location>
  <level>7</level>
  <rules_id>5712,5713</rules_id>
</active-response>
```

### Configuración para Monitoreo de Seguridad en Contenedores

Para mejorar la seguridad específica de contenedores:

```xml
<!-- Monitoreo de eventos de Docker -->
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/docker.log</location>
</localfile>

<!-- Monitoreo de cambios en imágenes y contenedores -->
<syscheck>
  <directories check_all="yes">/var/lib/docker/containers</directories>
  <directories check_all="yes">/var/lib/docker/image</directories>
</syscheck>

<!-- Monitoreo de comandos Docker -->
<localfile>
  <log_format>command</log_format>
  <command>docker events --format '{{json .}}'</command>
  <frequency>10</frequency>
</localfile>
```

## 6.6 Solución de Problemas Comunes

### Problema: El Agente No Se Conecta al Servidor

**Síntoma**: El agente muestra errores de conexión en los logs.

**Solución**:
1. Verifica que la dirección del servidor en `ossec.conf` es correcta:
   ```bash
   docker exec -it ubuntu-test cat /var/ossec/etc/ossec.conf | grep "<address>"
   ```
2. Asegúrate de que el puerto 1514 está abierto en el servidor:
   ```bash
   sudo ufw status | grep 1514
   ```
3. Verifica que el agente está correctamente registrado:
   ```bash
   docker exec -it ubuntu-test cat /var/ossec/etc/client.keys
   ```
4. Reinicia el agente:
   ```bash
   docker exec -it ubuntu-test /var/ossec/bin/wazuh-control restart
   ```

### Problema: Errores en el Registro del Agente

**Síntoma**: El comando `agent-auth` falla con errores.

**Solución**:
1. Verifica que el servicio de autenticación está habilitado en el servidor:
   ```bash
   sudo grep "<auth>" -A 15 /var/ossec/etc/ossec.conf
   ```
2. Asegúrate de que el puerto 1515 está abierto:
   ```bash
   sudo ufw status | grep 1515
   ```
3. Verifica la conectividad desde el contenedor al servidor:
   ```bash
   docker exec -it ubuntu-test ping IP_DEL_SERVIDOR_WAZUH
   ```
4. Intenta el registro con opciones de depuración:
   ```bash
   docker exec -it ubuntu-test /var/ossec/bin/agent-auth -m IP_DEL_SERVIDOR_WAZUH -p 1515 -d
   ```

### Problema: El Agente Se Desconecta Frecuentemente

**Síntoma**: El agente aparece como desconectado en el dashboard o en los logs del servidor.

**Solución**:
1. Verifica la estabilidad de la red entre el contenedor y el servidor:
   ```bash
   docker exec -it ubuntu-test ping -c 20 IP_DEL_SERVIDOR_WAZUH
   ```
2. Aumenta el tiempo de reconexión en la configuración del agente:
   ```xml
   <client>
     <time-reconnect>120</time-reconnect>
   </client>
   ```
3. Verifica si hay problemas de recursos en el contenedor:
   ```bash
   docker stats ubuntu-test
   ```

### Problema: No Se Reciben Eventos del Contenedor

**Síntoma**: El agente está conectado pero no se ven eventos en el dashboard.

**Solución**:
1. Verifica que los archivos de log existen y tienen los permisos correctos:
   ```bash
   docker exec -it ubuntu-test ls -la /var/log/
   ```
2. Genera algunos eventos de prueba:
   ```bash
   docker exec -it ubuntu-test logger "Evento de prueba para Wazuh"
   ```
3. Verifica la configuración de `localfile` en `ossec.conf`:
   ```bash
   docker exec -it ubuntu-test grep -A 5 "<localfile>" /var/ossec/etc/ossec.conf
   ```
4. Reinicia el agente después de modificar la configuración:
   ```bash
   docker exec -it ubuntu-test /var/ossec/bin/wazuh-control restart
   ```

Con estas configuraciones y soluciones, deberías tener el agente Wazuh funcionando correctamente en tus contenedores Docker. En la siguiente sección, exploraremos la implementación de monitoreo agentless para dispositivos que no pueden tener un agente instalado.
