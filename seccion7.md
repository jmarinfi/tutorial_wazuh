# 7. Implementación de Monitoreo Agentless

En las secciones anteriores, hemos explorado cómo instalar y configurar el agente Wazuh en contenedores Docker. Sin embargo, hay situaciones en las que no es posible o práctico instalar un agente en todos los dispositivos que queremos monitorear. Para estos casos, Wazuh ofrece una solución alternativa: el monitoreo agentless.

El monitoreo agentless permite supervisar dispositivos y sistemas sin necesidad de instalar un agente en ellos. En esta sección, explicaremos el concepto de monitoreo agentless, cómo configurarlo en el servidor Wazuh, y proporcionaremos ejemplos específicos para monitoreo de red y SSH.

## 7.1 Concepto de Monitoreo Agentless y Cuándo Utilizarlo

### ¿Qué es el Monitoreo Agentless?

El monitoreo agentless es una técnica que permite recopilar información de seguridad de dispositivos sin instalar software adicional en ellos. En lugar de depender de un agente instalado localmente, el servidor Wazuh se conecta directamente a los dispositivos utilizando protocolos existentes como SSH, y recopila la información necesaria.

### Ventajas del Monitoreo Agentless

- **No requiere instalación de software**: Ideal para dispositivos donde no se puede instalar software adicional.
- **Menor sobrecarga en los dispositivos**: Al no ejecutar un agente permanente, consume menos recursos.
- **Centralización**: Toda la lógica de monitoreo se gestiona desde el servidor Wazuh.
- **Flexibilidad**: Permite monitorear una amplia variedad de dispositivos, incluyendo routers, switches y otros dispositivos de red.

### Limitaciones del Monitoreo Agentless

- **Monitoreo no continuo**: Generalmente se ejecuta a intervalos programados, no en tiempo real.
- **Capacidades limitadas**: No ofrece todas las funcionalidades de un agente completo.
- **Requiere credenciales**: Necesita acceso SSH o similar a los dispositivos monitoreados.
- **Mayor carga en el servidor**: El servidor Wazuh debe manejar todas las conexiones y el procesamiento.

### Cuándo Utilizar Monitoreo Agentless

El monitoreo agentless es especialmente útil en los siguientes escenarios:

1. **Dispositivos de red**: Routers, switches, firewalls y otros dispositivos de red que no permiten la instalación de software de terceros.
2. **Sistemas embebidos**: Dispositivos con recursos limitados o sistemas operativos especializados.
3. **Dispositivos IoT**: Sensores y otros dispositivos IoT con capacidades limitadas.
4. **Sistemas críticos**: Servidores de producción donde se quiere minimizar el software instalado.
5. **Entornos heterogéneos**: Cuando se necesita monitorear una variedad de sistemas con diferentes arquitecturas y sistemas operativos.

## 7.2 Configuración en el Servidor Wazuh para Monitoreo Agentless

Para implementar el monitoreo agentless, necesitamos configurar el servidor Wazuh adecuadamente.

### Instalación de Dependencias

El monitoreo agentless de Wazuh utiliza principalmente SSH y requiere el paquete `expect` para la automatización de conexiones:

```bash
# Instalar el paquete expect
sudo apt update
sudo apt install -y expect
```

### Configuración Básica de Monitoreo Agentless

La configuración del monitoreo agentless se realiza en el archivo `ossec.conf` del servidor Wazuh:

```bash
# Editar el archivo de configuración
sudo nano /var/ossec/etc/ossec.conf
```

Añade la siguiente sección para habilitar el monitoreo agentless:

```xml
<agentless>
  <type>ssh_integrity_check_linux</type>
  <frequency>12h</frequency>
  <host>username@192.168.1.50</host>
  <state>periodic</state>
  <arguments>/etc /usr/bin /usr/sbin</arguments>
</agentless>
```

Esta configuración realizará una verificación de integridad de los directorios `/etc`, `/usr/bin` y `/usr/sbin` en el host `192.168.1.50` cada 12 horas.

### Tipos de Monitoreo Agentless

Wazuh ofrece varios tipos de monitoreo agentless:

1. **ssh_integrity_check_linux**: Verifica la integridad de archivos en sistemas Linux.
2. **ssh_integrity_check_bsd**: Verifica la integridad de archivos en sistemas BSD.
3. **ssh_generic_diff**: Compara la salida de un comando con la salida anterior.
4. **ssh_pixconfig_diff**: Específico para dispositivos Cisco PIX.

### Configuración de Múltiples Hosts

Puedes configurar múltiples hosts para monitoreo agentless:

```xml
<agentless>
  <type>ssh_integrity_check_linux</type>
  <frequency>12h</frequency>
  <host>username@192.168.1.50</host>
  <state>periodic</state>
  <arguments>/etc /usr/bin /usr/sbin</arguments>
</agentless>

<agentless>
  <type>ssh_generic_diff</type>
  <frequency>1h</frequency>
  <host>username@192.168.1.51</host>
  <state>periodic</state>
  <arguments>ls -la /var/log</arguments>
</agentless>
```

### Configuración de Autenticación SSH

Para que el monitoreo agentless funcione, el servidor Wazuh debe poder conectarse a los hosts remotos sin intervención manual. Esto se logra mediante la autenticación por clave SSH:

```bash
# Generar un par de claves SSH para el usuario ossec
sudo -u ossec ssh-keygen -t rsa -b 4096 -f /var/ossec/.ssh/id_rsa -N ""

# Mostrar la clave pública
sudo cat /var/ossec/.ssh/id_rsa.pub
```

Esta clave pública debe añadirse al archivo `~/.ssh/authorized_keys` de los usuarios en los hosts remotos.

### Registro de Hosts para Monitoreo Agentless

Wazuh proporciona un script para registrar hosts para monitoreo agentless:

```bash
# Registrar un host para monitoreo agentless
sudo /var/ossec/bin/agent_control -a
```

Este comando iniciará un asistente interactivo para añadir un nuevo host.

Alternativamente, puedes utilizar el script `register_host.sh`:

```bash
sudo /var/ossec/agentless/register_host.sh add username@192.168.1.50 password
```

> **Nota**: Almacenar contraseñas en texto plano no es recomendable para entornos de producción. Es preferible utilizar la autenticación por clave SSH.

## 7.3 Monitoreo de Red con Wazuh

El monitoreo de red es una parte importante de cualquier estrategia de seguridad. Wazuh puede ayudarnos a monitorear la actividad de red sin necesidad de agentes.

### Configuración de Sniffing de Red

Para monitorear el tráfico de red, podemos utilizar herramientas como `tcpdump` o `tshark` y enviar los resultados a Wazuh:

```xml
<agentless>
  <type>ssh_generic_diff</type>
  <frequency>10m</frequency>
  <host>username@192.168.1.50</host>
  <state>periodic</state>
  <arguments>tcpdump -n -c 100</arguments>
</agentless>
```

Esta configuración ejecutará `tcpdump` para capturar 100 paquetes cada 10 minutos y enviará los resultados a Wazuh.

### Monitoreo de Conexiones Activas

Podemos monitorear las conexiones activas en un host remoto:

```xml
<agentless>
  <type>ssh_generic_diff</type>
  <frequency>5m</frequency>
  <host>username@192.168.1.50</host>
  <state>periodic</state>
  <arguments>netstat -tulpn</arguments>
</agentless>
```

### Análisis de Tráfico con Reglas Personalizadas

Para analizar el tráfico capturado, podemos crear reglas personalizadas en Wazuh. Crea un archivo de reglas personalizado:

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

Añade reglas para detectar patrones sospechosos:

```xml
<group name="agentless,network,">
  <rule id="100200" level="7">
    <if_sid>530</if_sid>
    <match>^agentless: output: </match>
    <regex>port 22</regex>
    <description>SSH connection detected in network traffic</description>
  </rule>
  
  <rule id="100201" level="10">
    <if_sid>530</if_sid>
    <match>^agentless: output: </match>
    <regex>port 3389</regex>
    <description>RDP connection detected in network traffic</description>
  </rule>
</group>
```

Estas reglas generarán alertas cuando se detecten conexiones SSH o RDP en el tráfico capturado.

## 7.4 Monitoreo SSH para Dispositivos sin Agente

El monitoreo SSH es una de las formas más comunes de monitoreo agentless. Permite recopilar información de dispositivos remotos a través de conexiones SSH.

### Configuración de Monitoreo SSH Básico

Para configurar el monitoreo SSH básico:

```xml
<agentless>
  <type>ssh_generic_diff</type>
  <frequency>1h</frequency>
  <host>username@192.168.1.50</host>
  <state>periodic</state>
  <arguments>uname -a; uptime; who -a</arguments>
</agentless>
```

Esta configuración ejecutará los comandos `uname -a`, `uptime` y `who -a` cada hora y alertará sobre cualquier cambio en la salida.

### Monitoreo de Archivos de Log Remotos

Podemos monitorear archivos de log en dispositivos remotos:

```xml
<agentless>
  <type>ssh_generic_diff</type>
  <frequency>30m</frequency>
  <host>username@192.168.1.50</host>
  <state>periodic</state>
  <arguments>tail -n 50 /var/log/auth.log</arguments>
</agentless>
```

### Monitoreo de Cambios en Archivos de Configuración

Para detectar cambios en archivos de configuración críticos:

```xml
<agentless>
  <type>ssh_integrity_check_linux</type>
  <frequency>6h</frequency>
  <host>username@192.168.1.50</host>
  <state>periodic</state>
  <arguments>/etc/passwd /etc/shadow /etc/ssh/sshd_config</arguments>
</agentless>
```

### Monitoreo de Dispositivos de Red

Para dispositivos de red como routers o switches:

```xml
<agentless>
  <type>ssh_generic_diff</type>
  <frequency>12h</frequency>
  <host>admin@192.168.1.1</host>
  <state>periodic</state>
  <arguments>show running-config</arguments>
</agentless>
```

Esta configuración es específica para dispositivos Cisco y ejecutará el comando `show running-config` cada 12 horas.

### Autenticación y Seguridad

La seguridad es crucial cuando se implementa monitoreo SSH. Algunas consideraciones importantes:

1. **Utiliza autenticación por clave SSH**: Evita almacenar contraseñas en texto plano.
2. **Crea usuarios dedicados**: Utiliza usuarios con privilegios limitados para el monitoreo.
3. **Restringe los comandos permitidos**: Utiliza `command=` en el archivo `authorized_keys` para limitar los comandos que puede ejecutar el usuario.
4. **Utiliza conexiones cifradas**: Asegúrate de que las conexiones SSH utilizan algoritmos de cifrado fuertes.

Ejemplo de restricción de comandos en `authorized_keys`:

```
command="uname -a; uptime; who -a",no-port-forwarding,no-X11-forwarding,no-agent-forwarding ssh-rsa AAAAB3NzaC1yc2E...
```

## 7.5 Verificación del Monitoreo Agentless

Una vez configurado el monitoreo agentless, es importante verificar que está funcionando correctamente.

### Verificación Manual

Puedes ejecutar manualmente el monitoreo agentless para verificar su funcionamiento:

```bash
sudo /var/ossec/bin/agentless-entrypoint.sh
```

Este comando ejecutará todas las verificaciones agentless configuradas.

### Verificación de Logs

Verifica los logs del servidor Wazuh para asegurarte de que el monitoreo agentless está funcionando:

```bash
sudo tail -f /var/ossec/logs/ossec.log | grep agentless
```

Deberías ver mensajes relacionados con las verificaciones agentless.

### Verificación en el Dashboard

También puedes verificar el funcionamiento del monitoreo agentless a través del Wazuh dashboard:

1. Accede al dashboard en `https://IP_DEL_SERVIDOR_WAZUH`
2. Ve a la sección "Management" > "Status"
3. Busca información sobre el monitoreo agentless en la pestaña "Agents"

### Pruebas de Alertas

Para verificar que las alertas se generan correctamente, puedes realizar cambios en los sistemas monitoreados y comprobar si Wazuh los detecta:

1. Realiza un cambio en un archivo monitoreado
2. Espera a que se ejecute la próxima verificación agentless
3. Verifica los logs y el dashboard para confirmar que se ha generado una alerta

## 7.6 Casos de Uso Prácticos

Veamos algunos casos de uso prácticos para el monitoreo agentless.

### Caso 1: Monitoreo de Routers y Switches

```xml
<agentless>
  <type>ssh_generic_diff</type>
  <frequency>12h</frequency>
  <host>admin@192.168.1.1</host>
  <state>periodic</state>
  <arguments>show running-config</arguments>
</agentless>
```

Este caso de uso es ideal para detectar cambios no autorizados en la configuración de dispositivos de red.

### Caso 2: Monitoreo de Servidores Legacy

```xml
<agentless>
  <type>ssh_integrity_check_linux</type>
  <frequency>24h</frequency>
  <host>admin@192.168.1.100</host>
  <state>periodic</state>
  <arguments>/etc /bin /sbin</arguments>
</agentless>
```

Útil para servidores antiguos donde no se puede instalar el agente Wazuh.

### Caso 3: Monitoreo de Dispositivos IoT

```xml
<agentless>
  <type>ssh_generic_diff</type>
  <frequency>6h</frequency>
  <host>user@192.168.1.200</host>
  <state>periodic</state>
  <arguments>ps aux; netstat -tulpn</arguments>
</agentless>
```

Permite monitorear procesos y conexiones en dispositivos IoT.

### Caso 4: Auditoría de Cumplimiento

```xml
<agentless>
  <type>ssh_generic_diff</type>
  <frequency>168h</frequency> <!-- Una vez por semana -->
  <host>auditor@192.168.1.150</host>
  <state>periodic</state>
  <arguments>find /var/www -type f -name "*.php" -mtime -7 | xargs cat</arguments>
</agentless>
```

Este ejemplo busca y muestra el contenido de archivos PHP modificados en la última semana, útil para auditorías de cumplimiento.

## 7.7 Solución de Problemas Comunes

### Problema: Error de Autenticación SSH

**Síntoma**: Los logs muestran errores de autenticación SSH.

**Solución**:
1. Verifica que las claves SSH están correctamente configuradas:
   ```bash
   sudo -u ossec ssh -i /var/ossec/.ssh/id_rsa -o BatchMode=yes username@host
   ```
2. Asegúrate de que la clave pública está correctamente añadida al archivo `authorized_keys` del host remoto.
3. Verifica los permisos de los archivos de claves:
   ```bash
   sudo chmod 600 /var/ossec/.ssh/id_rsa
   sudo chmod 644 /var/ossec/.ssh/id_rsa.pub
   sudo chown -R ossec:ossec /var/ossec/.ssh
   ```

### Problema: Comandos No Ejecutados

**Síntoma**: No se ven resultados de los comandos configurados.

**Solución**:
1. Verifica que el usuario tiene permisos para ejecutar los comandos:
   ```bash
   sudo -u ossec ssh username@host "comando"
   ```
2. Comprueba si hay restricciones en el archivo `authorized_keys` del host remoto.
3. Verifica los logs para errores específicos:
   ```bash
   sudo grep "agentless" /var/ossec/logs/ossec.log
   ```

### Problema: No Se Generan Alertas

**Síntoma**: Se ejecutan los comandos pero no se generan alertas.

**Solución**:
1. Verifica que las reglas están correctamente configuradas:
   ```bash
   sudo grep -r "agentless" /var/ossec/etc/rules/
   ```
2. Asegúrate de que el nivel de alerta es adecuado:
   ```bash
   sudo grep -A 5 "<rule id=" /var/ossec/etc/rules/local_rules.xml
   ```
3. Comprueba si hay cambios en la salida de los comandos:
   ```bash
   sudo diff /var/ossec/agentless/previous_output/hostname_command /var/ossec/agentless/current_output/hostname_command
   ```

### Problema: Rendimiento Degradado

**Síntoma**: El servidor Wazuh muestra un rendimiento degradado después de configurar el monitoreo agentless.

**Solución**:
1. Reduce la frecuencia de las verificaciones:
   ```xml
   <frequency>24h</frequency> <!-- Cambiar a una frecuencia menor -->
   ```
2. Limita el número de hosts monitoreados simultáneamente.
3. Optimiza los comandos para que sean más eficientes:
   ```xml
   <arguments>find /etc -type f -mtime -1</arguments> <!-- Solo archivos modificados en el último día -->
   ```

Con estas configuraciones y soluciones, deberías poder implementar efectivamente el monitoreo agentless con Wazuh. En la siguiente sección, exploraremos cómo verificar y probar toda nuestra implementación de Wazuh.
