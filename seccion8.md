# 8. Verificación y Pruebas

Una vez que hemos implementado nuestra plataforma Wazuh completa, con el servidor, los agentes en contenedores y el monitoreo agentless, es fundamental verificar que todo funciona correctamente y realizar pruebas para asegurarnos de que el sistema detecta adecuadamente los eventos de seguridad.

En esta sección, aprenderemos a verificar la integridad de toda la instalación, generar eventos de prueba, interpretar los logs del sistema y solucionar problemas comunes.

## 8.1 Procedimientos para Verificar la Integridad de la Instalación

Antes de confiar en nuestro sistema Wazuh para la detección de amenazas, debemos asegurarnos de que todos los componentes están funcionando correctamente.

### Verificación del Estado de los Servicios

El primer paso es verificar que todos los servicios de Wazuh están en ejecución:

```bash
# Verificar el estado del Wazuh manager
sudo systemctl status wazuh-manager

# Verificar el estado del Wazuh indexer
sudo systemctl status wazuh-indexer

# Verificar el estado del Wazuh dashboard
sudo systemctl status wazuh-dashboard

# Verificar el estado de Filebeat
sudo systemctl status filebeat
```

Todos los servicios deberían mostrar el estado "active (running)".

### Verificación de la Comunicación entre Componentes

Es importante verificar que los componentes de Wazuh pueden comunicarse entre sí:

```bash
# Verificar la comunicación con el Wazuh indexer
curl -k -u admin:admin "https://localhost:9200/_cat/indices/wazuh-*?v&h=health,status,index,uuid,docs.count,docs.deleted,store.size,pri.store.size"

# Verificar que Filebeat está enviando datos al Wazuh indexer
sudo filebeat test output
```

### Verificación de los Agentes Conectados

Para verificar que los agentes están correctamente conectados al servidor:

```bash
# Listar todos los agentes
sudo /var/ossec/bin/agent_control -l

# Verificar el estado de un agente específico
sudo /var/ossec/bin/agent_control -i 001
```

También puedes verificar los agentes a través del Wazuh dashboard:

1. Accede al dashboard en `https://IP_DEL_SERVIDOR_WAZUH`
2. Ve a la sección "Agents"
3. Verifica que todos tus agentes aparecen como "Active"

### Verificación del Monitoreo Agentless

Para verificar que el monitoreo agentless está funcionando:

```bash
# Ejecutar manualmente el monitoreo agentless
sudo /var/ossec/bin/agentless-entrypoint.sh

# Verificar los logs relacionados con el monitoreo agentless
sudo grep "agentless" /var/ossec/logs/ossec.log
```

### Verificación de la Configuración

Es importante verificar que la configuración de Wazuh es correcta:

```bash
# Verificar la sintaxis de la configuración del Wazuh manager
sudo /var/ossec/bin/wazuh-logtest -t

# Verificar la configuración de Filebeat
sudo filebeat test config -c /etc/filebeat/filebeat.yml
```

## 8.2 Generación de Eventos de Prueba y Verificación de su Recepción

Una vez verificada la integridad de la instalación, es útil generar eventos de prueba para asegurarnos de que Wazuh los detecta y alerta correctamente.

### Generación de Eventos de Autenticación

Los eventos de autenticación son fáciles de generar y suelen ser monitoreados por Wazuh:

```bash
# En el servidor Wazuh o en un host con agente
# Intentar iniciar sesión con un usuario inexistente
ssh nonexistent_user@localhost
```

Este intento fallido de inicio de sesión debería generar una alerta en Wazuh.

### Generación de Eventos de Integridad de Archivos

Wazuh monitorea cambios en archivos importantes. Podemos generar estos eventos:

```bash
# En un host con agente Wazuh
# Crear un archivo en un directorio monitoreado
sudo touch /etc/test_file
echo "This is a test" | sudo tee /etc/test_file

# Modificar el archivo
echo "Modified content" | sudo tee /etc/test_file

# Eliminar el archivo
sudo rm /etc/test_file
```

### Generación de Eventos de Seguridad en Contenedores

Para probar la detección de eventos en contenedores:

```bash
# Acceder a un contenedor con agente Wazuh
docker exec -it ubuntu-wazuh bash

# Crear un usuario nuevo (acción que debería ser detectada)
useradd test_user

# Intentar ejecutar un comando con privilegios sin sudo
cat /etc/shadow

# Salir del contenedor
exit
```

### Simulación de Ataques Básicos

Para probar las capacidades de detección de Wazuh, podemos simular algunos ataques básicos:

```bash
# Simulación de escaneo de puertos (desde otro host)
nmap -p 1-1000 IP_DEL_SERVIDOR

# Simulación de fuerza bruta SSH (desde otro host)
# NOTA: Esto es solo para pruebas, no lo hagas en sistemas de producción sin autorización
hydra -l root -P /tmp/small_wordlist.txt ssh://IP_DEL_SERVIDOR -t 4
```

### Verificación de la Recepción de Eventos

Después de generar los eventos, debemos verificar que Wazuh los ha detectado:

```bash
# Verificar los logs de alertas
sudo tail -f /var/ossec/logs/alerts/alerts.log

# Buscar alertas específicas
sudo grep "sshd" /var/ossec/logs/alerts/alerts.log
```

También puedes verificar las alertas a través del Wazuh dashboard:

1. Accede al dashboard en `https://IP_DEL_SERVIDOR_WAZUH`
2. Ve a la sección "Security events"
3. Utiliza los filtros para encontrar los eventos que has generado

## 8.3 Interpretación de los Logs del Sistema para Diagnóstico

Los logs del sistema son una herramienta invaluable para diagnosticar problemas y entender el funcionamiento de Wazuh.

### Logs Principales de Wazuh

Wazuh genera varios archivos de log que contienen información importante:

```bash
# Log principal del Wazuh manager
sudo tail -f /var/ossec/logs/ossec.log

# Log de alertas
sudo tail -f /var/ossec/logs/alerts/alerts.log

# Log de Filebeat
sudo tail -f /var/log/filebeat/filebeat

# Log del Wazuh indexer
sudo tail -f /var/log/wazuh-indexer/wazuh-indexer.log

# Log del Wazuh dashboard
sudo tail -f /var/log/wazuh-dashboard/opensearch-dashboards.log
```

### Interpretación de Mensajes Comunes

Es importante entender los mensajes comunes que aparecen en los logs:

#### Mensajes del Wazuh Manager

- **Started**: Indica que un componente se ha iniciado correctamente.
- **Ended**: Indica que un componente se ha detenido correctamente.
- **Error**: Indica un problema que requiere atención.
- **Warning**: Indica un problema potencial que no impide el funcionamiento.
- **Info**: Información general sobre el funcionamiento del sistema.

#### Mensajes de Alertas

Las alertas en Wazuh tienen un formato específico:

```
** Alert 1586543278.123456: - pci_dss_10.2.4,pci_dss_10.2.5
2020 Apr 10 12:34:38 (ubuntu) 192.168.1.100->syscheck
Rule: 550 (level 7) -> 'Integrity checksum changed.'
Integrity checksum changed for: '/etc/passwd'
Size changed from '1675' to '1691'
Old md5sum was: '56f82d3c39743d9c6daa0d666c21e255'
New md5sum is : 'e2a5dca48aea2cf7a2a998b1c2231994'
Old sha1sum was: '2806dc8a0ad1a01d3c0f68bb23a6ce8c8b19e5d9'
New sha1sum is : 'f1a8a8d9d3e9b7077847b3c0c8d93a3b8b5560e2'
```

Esta alerta indica un cambio en el archivo `/etc/passwd`, mostrando los cambios en tamaño y checksums.

### Búsqueda Avanzada en Logs

Para análisis más detallados, podemos utilizar herramientas como `grep`, `awk` y `sed`:

```bash
# Buscar errores en el log principal
sudo grep "error" /var/ossec/logs/ossec.log

# Buscar alertas de nivel alto (nivel >= 10)
sudo grep -A 10 "level 1[0-9]" /var/ossec/logs/alerts/alerts.log

# Contar alertas por regla
sudo grep "Rule:" /var/ossec/logs/alerts/alerts.log | awk '{print $2}' | sort | uniq -c | sort -nr

# Extraer IPs de origen de alertas
sudo grep "Rule:" /var/ossec/logs/alerts/alerts.log | grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" | sort | uniq -c | sort -nr
```

### Rotación de Logs

Es importante entender cómo Wazuh maneja la rotación de logs para evitar que ocupen demasiado espacio:

```bash
# Verificar la configuración de rotación de logs
sudo cat /var/ossec/etc/internal_options.conf | grep "monitord.rotate"
```

Por defecto, Wazuh rota los logs diariamente y mantiene los últimos 7 días.

## 8.4 Procedimientos de Troubleshooting para Problemas Comunes

A pesar de una instalación cuidadosa, pueden surgir problemas. Aquí presentamos soluciones para los problemas más comunes.

### Problema: El Wazuh Manager No Inicia

**Síntomas**:
- El servicio wazuh-manager no inicia
- Errores en los logs

**Solución**:
1. Verifica la sintaxis de la configuración:
   ```bash
   sudo /var/ossec/bin/wazuh-logtest -t
   ```
2. Comprueba los logs para errores específicos:
   ```bash
   sudo tail -f /var/ossec/logs/ossec.log
   ```
3. Verifica los permisos de los archivos:
   ```bash
   sudo find /var/ossec -type f -name "*.xml" -exec chmod 640 {} \;
   sudo find /var/ossec -type d -exec chmod 750 {} \;
   sudo chown -R wazuh:wazuh /var/ossec
   ```
4. Reinicia el servicio:
   ```bash
   sudo systemctl restart wazuh-manager
   ```

### Problema: Agentes No Se Conectan al Servidor

**Síntomas**:
- Los agentes aparecen como "Disconnected" en el dashboard
- No se reciben eventos de los agentes

**Solución**:
1. Verifica la conectividad de red:
   ```bash
   # En el agente
   ping IP_DEL_SERVIDOR_WAZUH
   telnet IP_DEL_SERVIDOR_WAZUH 1514
   ```
2. Comprueba la configuración del agente:
   ```bash
   # En el agente
   cat /var/ossec/etc/ossec.conf | grep "<server>"
   ```
3. Verifica que el agente está registrado correctamente:
   ```bash
   # En el servidor
   sudo /var/ossec/bin/manage_agents -l
   ```
4. Reinicia el agente:
   ```bash
   # En el agente
   sudo /var/ossec/bin/wazuh-control restart
   ```

### Problema: El Wazuh Indexer No Funciona Correctamente

**Síntomas**:
- Errores al acceder al dashboard
- No se pueden ver alertas o eventos
- Errores en los logs del indexer

**Solución**:
1. Verifica el estado del servicio:
   ```bash
   sudo systemctl status wazuh-indexer
   ```
2. Comprueba los logs para errores específicos:
   ```bash
   sudo tail -f /var/log/wazuh-indexer/wazuh-indexer.log
   ```
3. Verifica la configuración:
   ```bash
   sudo cat /etc/wazuh-indexer/opensearch.yml
   ```
4. Comprueba el uso de memoria y disco:
   ```bash
   free -m
   df -h
   ```
5. Reinicia el servicio:
   ```bash
   sudo systemctl restart wazuh-indexer
   ```

### Problema: El Dashboard No Muestra Datos

**Síntomas**:
- El dashboard está accesible pero no muestra datos
- Aparecen errores de conexión con el indexer

**Solución**:
1. Verifica que el Wazuh indexer está funcionando:
   ```bash
   curl -k -u admin:admin "https://localhost:9200/_cat/indices/wazuh-*?v"
   ```
2. Comprueba la configuración del dashboard:
   ```bash
   sudo cat /etc/wazuh-dashboard/opensearch_dashboards.yml
   ```
3. Verifica los logs del dashboard:
   ```bash
   sudo tail -f /var/log/wazuh-dashboard/opensearch-dashboards.log
   ```
4. Reinicia los servicios:
   ```bash
   sudo systemctl restart wazuh-indexer
   sudo systemctl restart wazuh-dashboard
   ```

### Problema: Alertas No Se Generan para Eventos Específicos

**Síntomas**:
- Los eventos se registran pero no generan alertas
- No se ven alertas para ciertos tipos de eventos

**Solución**:
1. Verifica las reglas relacionadas:
   ```bash
   sudo grep -r "relevant_keyword" /var/ossec/etc/rules/
   ```
2. Comprueba el nivel de las reglas y la configuración de alertas:
   ```bash
   sudo cat /var/ossec/etc/ossec.conf | grep -A 10 "<alerts>"
   ```
3. Verifica que los decoders están funcionando correctamente:
   ```bash
   sudo /var/ossec/bin/wazuh-logtest -U "test log message"
   ```
4. Considera crear reglas personalizadas:
   ```bash
   sudo nano /var/ossec/etc/rules/local_rules.xml
   ```

### Problema: Rendimiento Degradado

**Síntomas**:
- El sistema se vuelve lento
- Alto uso de CPU o memoria
- Retrasos en la generación de alertas

**Solución**:
1. Verifica el uso de recursos:
   ```bash
   top
   htop
   ```
2. Comprueba el uso de disco:
   ```bash
   df -h
   du -sh /var/ossec/*
   ```
3. Optimiza la configuración del Wazuh indexer:
   ```bash
   sudo nano /etc/wazuh-indexer/opensearch.yml
   # Ajusta los valores de memoria según tus recursos
   # Ejemplo: -Xms4g -Xmx4g
   ```
4. Implementa políticas de retención de datos:
   ```bash
   # Configura la retención de índices
   curl -k -u admin:admin -X PUT "https://localhost:9200/_ilm/policy/wazuh_policy" -H 'Content-Type: application/json' -d'
   {
     "policy": {
       "phases": {
         "hot": {
           "actions": {}
         },
         "delete": {
           "min_age": "30d",
           "actions": {
             "delete": {}
           }
         }
       }
     }
   }'
   ```

## 8.5 Herramientas de Diagnóstico Adicionales

Además de los procedimientos básicos de troubleshooting, existen herramientas adicionales que pueden ayudarnos a diagnosticar problemas en nuestra instalación de Wazuh.

### Wazuh API

La API de Wazuh proporciona una forma programática de interactuar con el sistema y obtener información de diagnóstico:

```bash
# Obtener información del manager
curl -k -u admin:admin "https://localhost:55000/manager/status?pretty=true"

# Obtener información de los agentes
curl -k -u admin:admin "https://localhost:55000/agents?pretty=true"

# Verificar la configuración
curl -k -u admin:admin "https://localhost:55000/manager/configuration?pretty=true"
```

### Herramientas de Monitoreo del Sistema

Herramientas estándar de Linux pueden proporcionar información valiosa:

```bash
# Monitoreo de uso de CPU y memoria
htop

# Monitoreo de E/S de disco
iostat -x 1

# Monitoreo de red
iftop
netstat -tulpn

# Monitoreo de archivos abiertos
lsof -p $(pgrep -f wazuh-manager)
```

### Herramientas Específicas de Wazuh

Wazuh incluye herramientas específicas para diagnóstico:

```bash
# Verificar la base de datos de Wazuh
sudo /var/ossec/bin/wazuh-db-check

# Verificar la integridad de los archivos de Wazuh
sudo /var/ossec/bin/wazuh-control check-integrity

# Generar un diagnóstico completo
sudo /var/ossec/bin/wazuh-control diagnose
```

### Monitoreo de Logs en Tiempo Real

Para un monitoreo más efectivo, podemos utilizar herramientas que muestran los logs en tiempo real:

```bash
# Monitoreo de logs con colores
sudo apt install -y ccze
sudo tail -f /var/ossec/logs/ossec.log | ccze -A

# Monitoreo de múltiples logs simultáneamente
sudo apt install -y multitail
sudo multitail /var/ossec/logs/ossec.log /var/ossec/logs/alerts/alerts.log
```

## 8.6 Verificación de Cumplimiento y Auditoría

Además de verificar el funcionamiento técnico, es importante asegurarse de que nuestra implementación de Wazuh cumple con los requisitos de seguridad y cumplimiento normativo.

### Verificación de Políticas de Seguridad

Wazuh incluye módulos para verificar el cumplimiento de políticas de seguridad:

```bash
# Verificar el estado de SCA (Security Configuration Assessment)
curl -k -u admin:admin "https://localhost:55000/sca/001?pretty=true"
```

También puedes verificar el cumplimiento a través del dashboard:

1. Accede al dashboard en `https://IP_DEL_SERVIDOR_WAZUH`
2. Ve a la sección "Security configuration assessment"
3. Revisa los resultados para cada agente

### Auditoría de Acceso

Es importante auditar quién accede al sistema Wazuh:

```bash
# Verificar los logs de autenticación
sudo grep "authentication" /var/log/wazuh-dashboard/opensearch-dashboards.log

# Verificar los logs de la API
sudo grep "api" /var/ossec/logs/ossec.log
```

### Verificación de Cifrado y Comunicaciones Seguras

Asegúrate de que todas las comunicaciones están cifradas:

```bash
# Verificar la configuración de SSL/TLS
sudo grep -A 10 "<ssl>" /var/ossec/etc/ossec.conf

# Verificar los certificados
sudo openssl x509 -in /etc/wazuh-dashboard/certs/dashboard.pem -text -noout
```

### Pruebas de Penetración Básicas

Considera realizar pruebas de penetración básicas para verificar la seguridad:

```bash
# Escaneo de puertos
nmap -sV IP_DEL_SERVIDOR_WAZUH

# Verificación de configuraciones SSL/TLS
sslyze --regular IP_DEL_SERVIDOR_WAZUH:443
```

Con estos procedimientos de verificación y pruebas, deberías tener una buena comprensión del estado de tu implementación de Wazuh y estar preparado para solucionar cualquier problema que pueda surgir. En la siguiente sección, exploraremos consideraciones de seguridad adicionales y mejores prácticas para optimizar tu instalación de Wazuh.
