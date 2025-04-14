# 2. Instalación del Stack de Servidor Wazuh

En esta sección, vamos a realizar la instalación manual de los tres componentes principales del stack de servidor Wazuh: el Wazuh indexer, el servidor Wazuh y el Wazuh dashboard. También configuraremos los certificados SSL necesarios para asegurar las comunicaciones entre estos componentes.

## 2.1 Preparación del Sistema

Antes de comenzar con la instalación de los componentes de Wazuh, debemos preparar nuestro sistema Ubuntu. Asegúrate de estar utilizando un usuario con privilegios de administrador (sudo).

### Actualización del Sistema

Primero, actualicemos el sistema para asegurarnos de que todos los paquetes estén al día:

```bash
sudo apt update
sudo apt upgrade -y
```

### Instalación de Dependencias

Instalemos algunas dependencias necesarias:

```bash
sudo apt install -y curl apt-transport-https unzip wget libcap2-bin software-properties-common lsb-release gnupg
```

### Configuración de Límites del Sistema

Para un rendimiento óptimo, es recomendable ajustar algunos límites del sistema:

```bash
# Aumentar el número de archivos abiertos permitidos
echo "* soft nofile 65535" | sudo tee -a /etc/security/limits.conf
echo "* hard nofile 65535" | sudo tee -a /etc/security/limits.conf

# Aumentar el número de procesos permitidos
echo "* soft nproc 4096" | sudo tee -a /etc/security/limits.conf
echo "* hard nproc 4096" | sudo tee -a /etc/security/limits.conf

# Configurar el kernel para un mejor rendimiento con Elasticsearch
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## 2.2 Instalación del Wazuh Indexer

El Wazuh indexer es el componente responsable de indexar y almacenar las alertas y eventos generados por Wazuh. Está basado en Elasticsearch y optimizado para trabajar con Wazuh.

### Añadir el Repositorio de Wazuh

```bash
# Importar la clave GPG
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo apt-key add -

# Añadir el repositorio
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee -a /etc/apt/sources.list.d/wazuh.list

# Actualizar la información de paquetes
sudo apt update
```

### Instalación del Wazuh Indexer

```bash
sudo apt install -y wazuh-indexer
```

### Configuración Inicial del Wazuh Indexer

Editemos el archivo de configuración principal:

```bash
sudo nano /etc/wazuh-indexer/opensearch.yml
```

Modifica o añade las siguientes líneas:

```yaml
network.host: 0.0.0.0
node.name: node-1
cluster.initial_master_nodes: ["node-1"]
plugins.security.ssl.transport.pemcert_filepath: /etc/wazuh-indexer/certs/indexer.pem
plugins.security.ssl.transport.pemkey_filepath: /etc/wazuh-indexer/certs/indexer-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem
plugins.security.ssl.http.pemcert_filepath: /etc/wazuh-indexer/certs/indexer.pem
plugins.security.ssl.http.pemkey_filepath: /etc/wazuh-indexer/certs/indexer-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem
plugins.security.allow_default_init_securityindex: true
plugins.security.authcz.admin_dn:
  - "CN=admin,O=Wazuh,OU=Wazuh,L=California,C=US"
plugins.security.nodes_dn:
  - "CN=node-1,O=Wazuh,OU=Wazuh,L=California,C=US"
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
discovery.type: single-node
```

> **Nota**: La configuración anterior es para una instalación de un solo nodo. Para una configuración de clúster, necesitarás ajustar estos parámetros.

### Verificación del Servicio Wazuh Indexer

Iniciemos el servicio y verifiquemos su estado:

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-indexer
sudo systemctl start wazuh-indexer
sudo systemctl status wazuh-indexer
```

Esperamos unos momentos a que el servicio se inicie completamente y luego verificamos que esté funcionando correctamente:

```bash
curl -k https://localhost:9200
```

Deberías recibir una respuesta JSON que indica que el servicio está funcionando.

## 2.3 Instalación del Wazuh Server

El Wazuh server es el componente central que recibe y analiza los datos de los agentes, genera alertas y gestiona la configuración de los agentes.

### Instalación del Paquete Wazuh Manager

```bash
sudo apt install -y wazuh-manager
```

### Verificación del Servicio Wazuh Manager

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-manager
sudo systemctl start wazuh-manager
sudo systemctl status wazuh-manager
```

Para verificar que el servicio está funcionando correctamente, podemos comprobar los logs:

```bash
sudo tail -f /var/ossec/logs/ossec.log
```

Deberías ver mensajes que indican que el servicio se ha iniciado correctamente.

## 2.4 Instalación de Filebeat

Filebeat es necesario para enviar las alertas y eventos desde el Wazuh manager al Wazuh indexer.

### Instalación del Paquete Filebeat

```bash
sudo apt install -y filebeat
```

### Configuración de Filebeat para Wazuh

Descargamos la configuración predefinida de Filebeat para Wazuh:

```bash
sudo curl -so /etc/filebeat/filebeat.yml https://packages.wazuh.com/4.x/filebeat/filebeat.yml
```

Editamos el archivo de configuración para ajustar la conexión al Wazuh indexer:

```bash
sudo nano /etc/filebeat/filebeat.yml
```

Modifica la sección `output.elasticsearch` para que apunte a tu Wazuh indexer:

```yaml
output.elasticsearch:
  hosts: ["localhost:9200"]
  protocol: https
  username: ${username}
  password: ${password}
  ssl.certificate_authorities: ["/etc/filebeat/certs/root-ca.pem"]
  ssl.certificate: "/etc/filebeat/certs/filebeat.pem"
  ssl.key: "/etc/filebeat/certs/filebeat-key.pem"
```

### Configuración de Credenciales en Filebeat

Creamos un keystore para almacenar las credenciales de forma segura:

```bash
sudo filebeat keystore create
sudo echo "admin" | sudo filebeat keystore add username --stdin
sudo echo "admin" | sudo filebeat keystore add password --stdin
```

### Instalación del Módulo Wazuh para Filebeat

```bash
sudo filebeat modules enable wazuh
sudo curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-template.json | sudo tee /etc/filebeat/wazuh-template.json > /dev/null
sudo chmod go+r /etc/filebeat/wazuh-template.json
```

## 2.5 Instalación del Wazuh Dashboard

El Wazuh dashboard proporciona una interfaz web para visualizar y analizar los datos recopilados por Wazuh.

### Instalación del Paquete Wazuh Dashboard

```bash
sudo apt install -y wazuh-dashboard
```

### Configuración del Wazuh Dashboard

Editamos el archivo de configuración:

```bash
sudo nano /etc/wazuh-dashboard/opensearch_dashboards.yml
```

Modifica o añade las siguientes líneas:

```yaml
server.host: 0.0.0.0
server.port: 443
opensearch.hosts: ["https://localhost:9200"]
opensearch.ssl.verificationMode: certificate
opensearch.username: admin
opensearch.password: admin
opensearch.requestHeadersWhitelist: ["securitytenant", "Authorization"]
server.ssl.enabled: true
server.ssl.certificate: /etc/wazuh-dashboard/certs/dashboard.pem
server.ssl.key: /etc/wazuh-dashboard/certs/dashboard-key.pem
opensearch.ssl.certificateAuthorities: ["/etc/wazuh-dashboard/certs/root-ca.pem"]
uiSettings.overrides.defaultRoute: /app/wazuh
```

### Verificación del Servicio Wazuh Dashboard

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-dashboard
sudo systemctl start wazuh-dashboard
sudo systemctl status wazuh-dashboard
```

## 2.6 Configuración de Certificados SSL

Para asegurar las comunicaciones entre los componentes de Wazuh, necesitamos configurar certificados SSL. Wazuh proporciona una herramienta para generar estos certificados.

### Descarga de la Herramienta de Certificados

```bash
mkdir -p ~/wazuh-certs
cd ~/wazuh-certs
curl -sO https://packages.wazuh.com/4.x/wazuh-certs-tool.sh
chmod +x wazuh-certs-tool.sh
```

### Creación del Archivo de Configuración para los Certificados

Creamos un archivo `config.yml` con la configuración de nuestros nodos:

```bash
cat > config.yml << EOF
nodes:
  # Wazuh indexer nodes
  indexer:
    - name: node-1
      ip: <IP_DEL_SERVIDOR>
  
  # Wazuh server nodes
  server:
    - name: wazuh-server
      ip: <IP_DEL_SERVIDOR>
  
  # Wazuh dashboard nodes
  dashboard:
    - name: wazuh-dashboard
      ip: <IP_DEL_SERVIDOR>
EOF
```

Reemplaza `<IP_DEL_SERVIDOR>` con la dirección IP de tu servidor.

### Generación de los Certificados

```bash
./wazuh-certs-tool.sh -A
```

Este comando generará un archivo `wazuh-certificates.tar` que contiene todos los certificados necesarios.

### Distribución de los Certificados

Extraemos y distribuimos los certificados a los directorios correspondientes:

```bash
# Crear directorios para los certificados
sudo mkdir -p /etc/wazuh-indexer/certs
sudo mkdir -p /etc/wazuh-manager/certs
sudo mkdir -p /etc/filebeat/certs
sudo mkdir -p /etc/wazuh-dashboard/certs

# Extraer certificados
tar -xf wazuh-certificates.tar

# Copiar certificados para el Wazuh indexer
sudo cp -r ~/wazuh-certs/root-ca.pem ~/wazuh-certs/node-1.pem ~/wazuh-certs/node-1-key.pem /etc/wazuh-indexer/certs/
sudo mv /etc/wazuh-indexer/certs/node-1.pem /etc/wazuh-indexer/certs/indexer.pem
sudo mv /etc/wazuh-indexer/certs/node-1-key.pem /etc/wazuh-indexer/certs/indexer-key.pem

# Copiar certificados para el Wazuh server
sudo cp -r ~/wazuh-certs/root-ca.pem ~/wazuh-certs/wazuh-server.pem ~/wazuh-certs/wazuh-server-key.pem /etc/wazuh-manager/certs/
sudo mv /etc/wazuh-manager/certs/wazuh-server.pem /etc/wazuh-manager/certs/server.pem
sudo mv /etc/wazuh-manager/certs/wazuh-server-key.pem /etc/wazuh-manager/certs/server-key.pem

# Copiar certificados para Filebeat
sudo cp -r ~/wazuh-certs/root-ca.pem ~/wazuh-certs/wazuh-server.pem ~/wazuh-certs/wazuh-server-key.pem /etc/filebeat/certs/
sudo mv /etc/filebeat/certs/wazuh-server.pem /etc/filebeat/certs/filebeat.pem
sudo mv /etc/filebeat/certs/wazuh-server-key.pem /etc/filebeat/certs/filebeat-key.pem

# Copiar certificados para el Wazuh dashboard
sudo cp -r ~/wazuh-certs/root-ca.pem ~/wazuh-certs/wazuh-dashboard.pem ~/wazuh-certs/wazuh-dashboard-key.pem /etc/wazuh-dashboard/certs/
sudo mv /etc/wazuh-dashboard/certs/wazuh-dashboard.pem /etc/wazuh-dashboard/certs/dashboard.pem
sudo mv /etc/wazuh-dashboard/certs/wazuh-dashboard-key.pem /etc/wazuh-dashboard/certs/dashboard-key.pem

# Establecer permisos adecuados
sudo chmod 500 /etc/wazuh-indexer/certs /etc/wazuh-manager/certs /etc/filebeat/certs /etc/wazuh-dashboard/certs
sudo chmod 400 /etc/wazuh-indexer/certs/* /etc/wazuh-manager/certs/* /etc/filebeat/certs/* /etc/wazuh-dashboard/certs/*
sudo chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs
sudo chown -R wazuh:wazuh /etc/wazuh-manager/certs
sudo chown -R root:root /etc/filebeat/certs
sudo chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs
```

## 2.7 Configuración de la Conexión entre Componentes

### Configuración de la Conexión del Wazuh Manager con el Wazuh Indexer

Editamos el archivo de configuración del Wazuh manager:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Añadimos o modificamos la sección `<indexer>`:

```xml
<indexer>
  <enabled>yes</enabled>
  <hosts>
    <host>https://localhost:9200</host>
  </hosts>
  <ssl>
    <certificate_authorities>/etc/wazuh-manager/certs/root-ca.pem</certificate_authorities>
    <certificate>/etc/wazuh-manager/certs/server.pem</certificate>
    <key>/etc/wazuh-manager/certs/server-key.pem</key>
  </ssl>
</indexer>
```

### Reinicio de Servicios

Reiniciamos todos los servicios para aplicar los cambios:

```bash
sudo systemctl restart wazuh-indexer
sudo systemctl restart wazuh-manager
sudo systemctl restart filebeat
sudo systemctl restart wazuh-dashboard
```

## 2.8 Verificación de la Instalación

### Verificación del Wazuh Indexer

```bash
curl -k -u admin:admin "https://localhost:9200/_cat/indices/wazuh-*?v&h=health,status,index,uuid,docs.count,docs.deleted,store.size,pri.store.size"
```

Deberías ver una lista de índices de Wazuh.

### Verificación del Wazuh Manager

```bash
sudo /var/ossec/bin/wazuh-control status
```

Todos los servicios deberían estar en estado "running".

### Verificación de Filebeat

```bash
sudo filebeat test output
```

Deberías ver un mensaje indicando que la conexión con Elasticsearch es exitosa.

### Verificación del Wazuh Dashboard

Abre un navegador web y accede a la dirección `https://<IP_DEL_SERVIDOR>`. Deberías ver la página de inicio de sesión del Wazuh dashboard. Las credenciales por defecto son:

- Usuario: admin
- Contraseña: admin

> **Importante**: Cambia estas credenciales por defecto inmediatamente después de verificar que puedes acceder al dashboard.

## 2.9 Solución de Problemas Comunes

### Problema: El Wazuh Indexer no Inicia

Verifica los logs:

```bash
sudo tail -f /var/log/wazuh-indexer/wazuh-indexer.log
```

Posibles soluciones:
- Asegúrate de que los certificados están correctamente configurados
- Verifica que los permisos de los archivos de certificados son correctos
- Comprueba que el valor de `vm.max_map_count` es al menos 262144

### Problema: El Wazuh Manager no Inicia

Verifica los logs:

```bash
sudo tail -f /var/ossec/logs/ossec.log
```

Posibles soluciones:
- Comprueba la sintaxis del archivo `ossec.conf`
- Verifica que los certificados están correctamente configurados
- Asegúrate de que los permisos de los directorios y archivos son correctos

### Problema: Filebeat no Puede Conectar con el Wazuh Indexer

Verifica los logs:

```bash
sudo tail -f /var/log/filebeat/filebeat
```

Posibles soluciones:
- Comprueba que el Wazuh indexer está funcionando
- Verifica que las credenciales en el keystore son correctas
- Asegúrate de que los certificados están correctamente configurados

### Problema: No Puedo Acceder al Wazuh Dashboard

Verifica los logs:

```bash
sudo tail -f /var/log/wazuh-dashboard/opensearch-dashboards.log
```

Posibles soluciones:
- Comprueba que el servicio está en ejecución
- Verifica que el firewall permite conexiones al puerto 443
- Asegúrate de que los certificados están correctamente configurados

Con esto, hemos completado la instalación y configuración del stack de servidor Wazuh. En la siguiente sección, configuraremos el acceso remoto al dashboard de Wazuh.
