# 5. Creación y Configuración de Contenedores para Pruebas

Una vez que hemos instalado y configurado Docker en nuestro servidor, el siguiente paso es crear contenedores de prueba que utilizaremos para demostrar el monitoreo con Wazuh. En esta sección, aprenderemos a crear y configurar contenedores Docker, gestionar la persistencia de datos, configurar redes y verificar el estado y los logs de los contenedores.

## 5.1 Creación de Contenedores Ubuntu/Debian

Para nuestras pruebas, crearemos contenedores basados en Ubuntu y Debian, que son distribuciones Linux ampliamente utilizadas y compatibles con el agente Wazuh.

### Creación de un Contenedor Ubuntu

Vamos a crear un contenedor Ubuntu con las opciones necesarias para nuestras pruebas:

```bash
docker run -d \
  --name ubuntu-test \
  --hostname ubuntu-test \
  --network wazuh-net \
  -p 8080:80 \
  -v ubuntu-data:/var/data \
  --restart unless-stopped \
  ubuntu:22.04 \
  sleep infinity
```

Explicación de las opciones utilizadas:
- `-d`: Ejecuta el contenedor en segundo plano (modo detached)
- `--name ubuntu-test`: Asigna el nombre "ubuntu-test" al contenedor
- `--hostname ubuntu-test`: Establece el hostname del contenedor
- `--network wazuh-net`: Conecta el contenedor a la red "wazuh-net" que creamos anteriormente
- `-p 8080:80`: Mapea el puerto 80 del contenedor al puerto 8080 del host
- `-v ubuntu-data:/var/data`: Crea y monta un volumen llamado "ubuntu-data" en la ruta /var/data del contenedor
- `--restart unless-stopped`: Configura el contenedor para que se reinicie automáticamente a menos que se detenga manualmente
- `ubuntu:22.04`: Utiliza la imagen Ubuntu 22.04
- `sleep infinity`: Mantiene el contenedor en ejecución indefinidamente

### Creación de un Contenedor Debian

De manera similar, vamos a crear un contenedor Debian:

```bash
docker run -d \
  --name debian-test \
  --hostname debian-test \
  --network wazuh-net \
  -p 8081:80 \
  -v debian-data:/var/data \
  --restart unless-stopped \
  debian:11 \
  sleep infinity
```

### Verificación de los Contenedores Creados

Para verificar que los contenedores se han creado correctamente:

```bash
docker ps
```

Deberías ver ambos contenedores en la lista con el estado "Up".

## 5.2 Configuración de Persistencia de Datos

La persistencia de datos es crucial para mantener la información importante incluso cuando los contenedores se detienen o se eliminan. Docker ofrece varias opciones para la persistencia de datos.

### Volúmenes de Docker

Los volúmenes de Docker son el mecanismo preferido para la persistencia de datos. Son completamente gestionados por Docker y ofrecen mejor rendimiento que los bind mounts.

#### Creación de Volúmenes

Ya hemos creado volúmenes implícitamente en los comandos anteriores, pero también podemos crearlos explícitamente:

```bash
# Crear un nuevo volumen
docker volume create wazuh-logs

# Listar volúmenes existentes
docker volume ls
```

#### Inspección de Volúmenes

Para obtener información detallada sobre un volumen:

```bash
docker volume inspect ubuntu-data
```

Esto mostrará información como la ubicación del volumen en el sistema de archivos del host.

#### Montaje de Volúmenes en Contenedores Existentes

Si necesitas montar un volumen en un contenedor que ya está en ejecución, debes recrear el contenedor:

```bash
# Detener y eliminar el contenedor existente
docker stop ubuntu-test
docker rm ubuntu-test

# Recrear el contenedor con el volumen adicional
docker run -d \
  --name ubuntu-test \
  --hostname ubuntu-test \
  --network wazuh-net \
  -p 8080:80 \
  -v ubuntu-data:/var/data \
  -v wazuh-logs:/var/log/wazuh \
  --restart unless-stopped \
  ubuntu:22.04 \
  sleep infinity
```

### Bind Mounts

Los bind mounts permiten montar directorios o archivos específicos del host en el contenedor. Son útiles cuando necesitas acceder a archivos específicos del host.

```bash
# Crear un directorio en el host para los logs
mkdir -p ~/docker-logs

# Ejecutar un contenedor con un bind mount
docker run -d \
  --name nginx-test \
  --hostname nginx-test \
  --network wazuh-net \
  -p 8082:80 \
  -v ~/docker-logs:/var/log/nginx \
  --restart unless-stopped \
  nginx:latest
```

### tmpfs Mounts

Los montajes tmpfs almacenan datos en memoria, lo que es útil para información temporal o sensible que no debe persistir:

```bash
docker run -d \
  --name redis-test \
  --hostname redis-test \
  --network wazuh-net \
  -p 6379:6379 \
  --tmpfs /tmp \
  --restart unless-stopped \
  redis:latest
```

## 5.3 Configuración de Red

La configuración de red adecuada es esencial para que los contenedores puedan comunicarse entre sí y con el mundo exterior.

### Tipos de Redes en Docker

Docker ofrece varios tipos de redes:

1. **Bridge**: Es el tipo de red predeterminado. Los contenedores en una red bridge pueden comunicarse entre sí y con el host.
2. **Host**: Los contenedores comparten la pila de red del host, eliminando el aislamiento de red.
3. **Overlay**: Permite la comunicación entre contenedores en diferentes hosts Docker, útil para clústeres.
4. **Macvlan**: Asigna direcciones MAC a contenedores, haciéndolos aparecer como dispositivos físicos en la red.
5. **None**: Deshabilita la red para el contenedor.

### Creación de Redes Personalizadas

Ya hemos creado una red bridge personalizada llamada "wazuh-net". Podemos crear redes adicionales según sea necesario:

```bash
# Crear una red bridge adicional
docker network create --driver bridge app-net

# Crear una red con un rango de IP específico
docker network create --driver bridge --subnet=172.20.0.0/16 --gateway=172.20.0.1 custom-net
```

### Conexión de Contenedores a Múltiples Redes

Un contenedor puede estar conectado a múltiples redes:

```bash
# Conectar un contenedor existente a una red adicional
docker network connect app-net ubuntu-test

# Verificar las redes de un contenedor
docker inspect -f '{{range $key, $value := .NetworkSettings.Networks}}{{$key}} {{end}}' ubuntu-test
```

### Configuración de DNS

Docker configura automáticamente el DNS para que los contenedores puedan resolverse por nombre. Podemos personalizar esta configuración:

```bash
# Ejecutar un contenedor con configuración DNS personalizada
docker run -d \
  --name custom-dns \
  --hostname custom-dns \
  --dns 8.8.8.8 \
  --dns 8.8.4.4 \
  --dns-search example.com \
  ubuntu:22.04 \
  sleep infinity
```

## 5.4 Verificación y Monitoreo de Contenedores

Es importante poder verificar el estado y monitorear los logs de los contenedores para asegurarse de que están funcionando correctamente.

### Verificación del Estado de los Contenedores

```bash
# Listar todos los contenedores en ejecución
docker ps

# Listar todos los contenedores (incluyendo los detenidos)
docker ps -a

# Obtener información detallada de un contenedor
docker inspect ubuntu-test
```

### Monitoreo de Recursos

Para monitorear el uso de recursos de los contenedores:

```bash
# Ver estadísticas en tiempo real
docker stats

# Ver estadísticas de un contenedor específico
docker stats ubuntu-test
```

### Acceso a los Logs del Contenedor

```bash
# Ver los logs de un contenedor
docker logs ubuntu-test

# Ver los logs en tiempo real
docker logs -f ubuntu-test

# Ver las últimas 100 líneas de logs
docker logs --tail 100 ubuntu-test
```

### Ejecución de Comandos en Contenedores en Ejecución

Para interactuar con un contenedor en ejecución:

```bash
# Ejecutar un comando en un contenedor
docker exec ubuntu-test ls -la /var/data

# Obtener una shell interactiva
docker exec -it ubuntu-test bash
```

## 5.5 Configuración de Contenedores para Aplicaciones Específicas

Ahora, vamos a configurar algunos contenedores con aplicaciones específicas que serán útiles para nuestras pruebas con Wazuh.

### Contenedor con Servidor Web (Nginx)

```bash
# Crear un contenedor con Nginx
docker run -d \
  --name nginx-server \
  --hostname nginx-server \
  --network wazuh-net \
  -p 8083:80 \
  -v nginx-data:/usr/share/nginx/html \
  --restart unless-stopped \
  nginx:latest
```

Para personalizar la página web:

```bash
# Crear un archivo HTML personalizado
echo "<html><body><h1>Servidor de prueba para Wazuh</h1></body></html>" > index.html

# Copiar el archivo al contenedor
docker cp index.html nginx-server:/usr/share/nginx/html/index.html
```

### Contenedor con Base de Datos (MySQL)

```bash
# Crear un contenedor con MySQL
docker run -d \
  --name mysql-server \
  --hostname mysql-server \
  --network wazuh-net \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=wazuh-test-password \
  -e MYSQL_DATABASE=wazuh_test \
  -e MYSQL_USER=wazuh \
  -e MYSQL_PASSWORD=wazuh \
  -v mysql-data:/var/lib/mysql \
  --restart unless-stopped \
  mysql:8.0
```

### Contenedor con Aplicación Vulnerable para Pruebas

Para probar las capacidades de detección de Wazuh, podemos crear un contenedor con una aplicación intencionalmente vulnerable:

```bash
# Crear un contenedor con DVWA (Damn Vulnerable Web Application)
docker run -d \
  --name dvwa \
  --hostname dvwa \
  --network wazuh-net \
  -p 8084:80 \
  --restart unless-stopped \
  vulnerables/web-dvwa
```

## 5.6 Gestión del Ciclo de Vida de los Contenedores

Es importante saber cómo gestionar el ciclo de vida completo de los contenedores.

### Inicio y Detención de Contenedores

```bash
# Detener un contenedor
docker stop ubuntu-test

# Iniciar un contenedor detenido
docker start ubuntu-test

# Reiniciar un contenedor
docker restart ubuntu-test
```

### Pausa y Reanudación de Contenedores

```bash
# Pausar un contenedor (congelar sus procesos)
docker pause ubuntu-test

# Reanudar un contenedor pausado
docker unpause ubuntu-test
```

### Eliminación de Contenedores

```bash
# Eliminar un contenedor detenido
docker rm ubuntu-test

# Forzar la eliminación de un contenedor en ejecución
docker rm -f ubuntu-test

# Eliminar todos los contenedores detenidos
docker container prune
```

### Actualización de Contenedores

Para actualizar un contenedor a una nueva imagen:

```bash
# Extraer la última versión de la imagen
docker pull ubuntu:latest

# Recrear el contenedor con la nueva imagen
docker stop ubuntu-test
docker rm ubuntu-test
docker run -d \
  --name ubuntu-test \
  --hostname ubuntu-test \
  --network wazuh-net \
  -p 8080:80 \
  -v ubuntu-data:/var/data \
  --restart unless-stopped \
  ubuntu:latest \
  sleep infinity
```

## 5.7 Uso de Docker Compose para Gestionar Múltiples Contenedores

Docker Compose facilita la gestión de aplicaciones multi-contenedor. Vamos a crear un archivo `docker-compose.yml` que defina todos nuestros contenedores de prueba:

```bash
# Crear un directorio para el proyecto
mkdir -p ~/wazuh-docker-test
cd ~/wazuh-docker-test

# Crear el archivo docker-compose.yml
nano docker-compose.yml
```

Añade el siguiente contenido al archivo:

```yaml
version: '3'

services:
  ubuntu-test:
    image: ubuntu:22.04
    container_name: ubuntu-test
    hostname: ubuntu-test
    command: sleep infinity
    restart: unless-stopped
    networks:
      - wazuh-net
    ports:
      - "8080:80"
    volumes:
      - ubuntu-data:/var/data

  debian-test:
    image: debian:11
    container_name: debian-test
    hostname: debian-test
    command: sleep infinity
    restart: unless-stopped
    networks:
      - wazuh-net
    ports:
      - "8081:80"
    volumes:
      - debian-data:/var/data

  nginx-server:
    image: nginx:latest
    container_name: nginx-server
    hostname: nginx-server
    restart: unless-stopped
    networks:
      - wazuh-net
    ports:
      - "8083:80"
    volumes:
      - nginx-data:/usr/share/nginx/html

  mysql-server:
    image: mysql:8.0
    container_name: mysql-server
    hostname: mysql-server
    restart: unless-stopped
    networks:
      - wazuh-net
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: wazuh-test-password
      MYSQL_DATABASE: wazuh_test
      MYSQL_USER: wazuh
      MYSQL_PASSWORD: wazuh
    volumes:
      - mysql-data:/var/lib/mysql

  dvwa:
    image: vulnerables/web-dvwa
    container_name: dvwa
    hostname: dvwa
    restart: unless-stopped
    networks:
      - wazuh-net
    ports:
      - "8084:80"

networks:
  wazuh-net:
    external: true

volumes:
  ubuntu-data:
  debian-data:
  nginx-data:
  mysql-data:
```

### Iniciar los Contenedores con Docker Compose

```bash
# Iniciar todos los servicios definidos en docker-compose.yml
docker compose up -d

# Verificar el estado de los servicios
docker compose ps
```

### Gestión de Servicios con Docker Compose

```bash
# Detener todos los servicios
docker compose stop

# Iniciar todos los servicios
docker compose start

# Reiniciar todos los servicios
docker compose restart

# Detener y eliminar todos los contenedores, redes y volúmenes
docker compose down

# Detener y eliminar todos los contenedores y redes, pero conservar los volúmenes
docker compose down --volumes
```

## 5.8 Solución de Problemas Comunes

### Problema: El Contenedor se Detiene Inmediatamente

**Síntoma**: El contenedor se crea pero se detiene inmediatamente.

**Solución**:
1. Verifica los logs del contenedor:
   ```bash
   docker logs <nombre_contenedor>
   ```
2. Asegúrate de que el comando principal del contenedor no termina inmediatamente. Para contenedores basados en Ubuntu/Debian que solo necesitas mantener en ejecución, usa `sleep infinity` como comando.
3. Verifica si hay errores en la configuración del contenedor:
   ```bash
   docker inspect <nombre_contenedor>
   ```

### Problema: No se Puede Acceder al Servicio Expuesto

**Síntoma**: No puedes acceder a un servicio expuesto a través de un puerto mapeado.

**Solución**:
1. Verifica que el contenedor está en ejecución:
   ```bash
   docker ps
   ```
2. Comprueba que el mapeo de puertos es correcto:
   ```bash
   docker port <nombre_contenedor>
   ```
3. Asegúrate de que el servicio dentro del contenedor está escuchando en la interfaz correcta (0.0.0.0 o la IP interna del contenedor).
4. Verifica si hay reglas de firewall que bloquean el acceso:
   ```bash
   sudo ufw status
   ```

### Problema: Problemas de Red entre Contenedores

**Síntoma**: Los contenedores no pueden comunicarse entre sí.

**Solución**:
1. Asegúrate de que los contenedores están en la misma red:
   ```bash
   docker network inspect wazuh-net
   ```
2. Verifica que los nombres de host se resuelven correctamente:
   ```bash
   docker exec ubuntu-test ping debian-test
   ```
3. Comprueba si hay conflictos de IP:
   ```bash
   docker network inspect wazuh-net | grep IPv4Address
   ```

### Problema: Pérdida de Datos al Recrear Contenedores

**Síntoma**: Los datos se pierden al recrear un contenedor.

**Solución**:
1. Asegúrate de utilizar volúmenes para datos persistentes:
   ```bash
   docker volume ls
   ```
2. Verifica que los volúmenes están correctamente montados:
   ```bash
   docker inspect -f '{{ .Mounts }}' <nombre_contenedor>
   ```
3. Si utilizas bind mounts, asegúrate de que las rutas del host existen y tienen los permisos adecuados.

Con estos contenedores de prueba configurados, ahora tenemos un entorno ideal para implementar y probar el agente Wazuh en contenedores Docker. En la siguiente sección, aprenderemos a instalar y configurar el agente Wazuh en estos contenedores.
