# 3. Configuración de Acceso Remoto al Dashboard

Una vez que hemos instalado y configurado correctamente el stack de servidor Wazuh, el siguiente paso es configurar el acceso remoto al Wazuh dashboard. Esto nos permitirá administrar y monitorear nuestra plataforma Wazuh desde cualquier ubicación, no solo desde el servidor donde está instalado.

En esta sección, exploraremos diferentes métodos para configurar el acceso remoto, incluyendo la configuración directa, el uso de túneles SSH, y consideraciones importantes de seguridad para proteger nuestro dashboard.

## 3.1 Configuración de Acceso Directo

Por defecto, el Wazuh dashboard está configurado para escuchar en todas las interfaces de red (`0.0.0.0`) en el puerto 443 (HTTPS). Sin embargo, es posible que necesitemos realizar algunos ajustes adicionales para garantizar un acceso remoto seguro y sin problemas.

### Verificación de la Configuración Actual

Primero, verifiquemos la configuración actual del Wazuh dashboard:

```bash
sudo cat /etc/wazuh-dashboard/opensearch_dashboards.yml | grep -E 'server.host|server.port'
```

Deberías ver algo como:

```
server.host: 0.0.0.0
server.port: 443
```

Si la configuración es diferente, edita el archivo para establecer estos valores:

```bash
sudo nano /etc/wazuh-dashboard/opensearch_dashboards.yml
```

Asegúrate de que las siguientes líneas estén presentes y correctamente configuradas:

```yaml
server.host: 0.0.0.0
server.port: 443
server.ssl.enabled: true
server.ssl.certificate: /etc/wazuh-dashboard/certs/dashboard.pem
server.ssl.key: /etc/wazuh-dashboard/certs/dashboard-key.pem
```

Guarda los cambios y reinicia el servicio:

```bash
sudo systemctl restart wazuh-dashboard
```

### Configuración del Firewall

Para permitir el acceso remoto al dashboard, debemos asegurarnos de que el firewall permita conexiones al puerto 443:

```bash
sudo ufw allow 443/tcp
sudo ufw status
```

### Acceso al Dashboard

Ahora deberías poder acceder al Wazuh dashboard desde cualquier navegador web utilizando la siguiente URL:

```
https://<IP_DEL_SERVIDOR>
```

Reemplaza `<IP_DEL_SERVIDOR>` con la dirección IP pública o el nombre de dominio de tu servidor.

> **Nota**: Al acceder por primera vez, es posible que recibas una advertencia de seguridad en tu navegador debido al uso de un certificado SSL autofirmado. Puedes proceder de manera segura aceptando el riesgo o, para entornos de producción, considera la instalación de un certificado SSL válido emitido por una autoridad de certificación reconocida.

## 3.2 Configuración de Túnel SSH

Si prefieres no exponer directamente el puerto 443 a Internet o si estás en un entorno donde el acceso directo no es posible debido a restricciones de red, puedes utilizar un túnel SSH para acceder de forma segura al Wazuh dashboard.

### Creación de un Túnel SSH Temporal

Desde tu máquina local, puedes crear un túnel SSH temporal con el siguiente comando:

```bash
ssh -L 8443:localhost:443 usuario@ip_servidor_remoto
```

Por ejemplo:

```bash
ssh -L 8443:localhost:443 ubuntu@192.168.1.100
```

Este comando crea un túnel que reenvía el puerto local 8443 al puerto 443 del servidor remoto. Mientras mantengas abierta esta conexión SSH, podrás acceder al Wazuh dashboard a través de tu navegador local utilizando la URL:

```
https://localhost:8443
```

### Configuración de un Túnel SSH Persistente

Para crear un túnel SSH más persistente, puedes utilizar la configuración del archivo `~/.ssh/config` en tu máquina local:

1. Edita o crea el archivo de configuración SSH:

```bash
nano ~/.ssh/config
```

2. Añade la siguiente configuración:

```
Host wazuh-tunnel
    HostName <IP_DEL_SERVIDOR>
    User <USUARIO>
    LocalForward 8443 localhost:443
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

Reemplaza `<IP_DEL_SERVIDOR>` y `<USUARIO>` con los valores correspondientes.

3. Guarda el archivo y establece los permisos adecuados:

```bash
chmod 600 ~/.ssh/config
```

4. Ahora puedes establecer el túnel simplemente con:

```bash
ssh wazuh-tunnel
```

Y acceder al dashboard a través de `https://localhost:8443`.

### Uso de AutoSSH para Túneles Persistentes

Para túneles aún más robustos que se reconecten automáticamente en caso de fallos, puedes utilizar AutoSSH:

1. Instala AutoSSH en tu máquina local:

```bash
sudo apt install -y autossh
```

2. Crea un script para iniciar el túnel:

```bash
nano ~/wazuh-tunnel.sh
```

3. Añade el siguiente contenido:

```bash
#!/bin/bash
autossh -M 0 -o "ServerAliveInterval 30" -o "ServerAliveCountMax 3" -L 8443:localhost:443 usuario@ip_servidor_remoto -N
```

4. Haz el script ejecutable:

```bash
chmod +x ~/wazuh-tunnel.sh
```

5. Para iniciar el túnel:

```bash
~/wazuh-tunnel.sh
```

Para que el túnel se inicie automáticamente al arrancar tu sistema, puedes añadirlo a cron:

```bash
(crontab -l 2>/dev/null; echo "@reboot ~/wazuh-tunnel.sh") | crontab -
```

## 3.3 Consideraciones de Seguridad para el Acceso Remoto

Al configurar el acceso remoto al Wazuh dashboard, es crucial implementar medidas de seguridad adecuadas para proteger tu infraestructura.

### Implementación de Autenticación de Dos Factores

Para añadir una capa adicional de seguridad, puedes implementar la autenticación de dos factores (2FA) utilizando herramientas como Google Authenticator.

1. Instala los paquetes necesarios en el servidor:

```bash
sudo apt install -y libpam-google-authenticator nginx
```

2. Configura Nginx como proxy inverso con autenticación básica:

```bash
sudo nano /etc/nginx/sites-available/wazuh-dashboard
```

3. Añade la siguiente configuración:

```nginx
server {
    listen 443 ssl;
    server_name _;

    ssl_certificate /etc/wazuh-dashboard/certs/dashboard.pem;
    ssl_certificate_key /etc/wazuh-dashboard/certs/dashboard-key.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;

    auth_basic "Wazuh Dashboard";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass https://localhost:443;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

4. Crea un archivo de contraseñas para la autenticación básica:

```bash
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

5. Habilita el sitio y reinicia Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/wazuh-dashboard /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

6. Configura Google Authenticator para el usuario que accederá al dashboard:

```bash
google-authenticator
```

Sigue las instrucciones en pantalla para configurar 2FA.

### Limitación de Intentos de Acceso

Para proteger contra ataques de fuerza bruta, puedes implementar la limitación de intentos de acceso utilizando Fail2Ban:

1. Instala Fail2Ban si aún no lo has hecho:

```bash
sudo apt install -y fail2ban
```

2. Crea una configuración personalizada para proteger el acceso al Wazuh dashboard:

```bash
sudo nano /etc/fail2ban/jail.d/wazuh-dashboard.conf
```

3. Añade la siguiente configuración:

```
[wazuh-dashboard]
enabled = true
port = 443
filter = wazuh-dashboard
logpath = /var/log/nginx/access.log
maxretry = 5
bantime = 3600
```

4. Crea un filtro personalizado:

```bash
sudo nano /etc/fail2ban/filter.d/wazuh-dashboard.conf
```

5. Añade la siguiente configuración:

```
[Definition]
failregex = ^<HOST> - .* "GET /.*" 401
ignoreregex =
```

6. Reinicia Fail2Ban:

```bash
sudo systemctl restart fail2ban
```

### Uso de Certificados SSL Válidos

Para entornos de producción, es altamente recomendable utilizar certificados SSL válidos emitidos por una autoridad de certificación reconocida. Puedes utilizar Let's Encrypt para obtener certificados gratuitos:

1. Instala Certbot:

```bash
sudo apt install -y certbot python3-certbot-nginx
```

2. Obtén un certificado (asegúrate de tener un nombre de dominio apuntando a tu servidor):

```bash
sudo certbot --nginx -d tu-dominio.com
```

3. Sigue las instrucciones en pantalla para completar el proceso.

4. Actualiza la configuración del Wazuh dashboard para utilizar los nuevos certificados:

```bash
sudo nano /etc/wazuh-dashboard/opensearch_dashboards.yml
```

5. Modifica las rutas de los certificados:

```yaml
server.ssl.certificate: /etc/letsencrypt/live/tu-dominio.com/fullchain.pem
server.ssl.key: /etc/letsencrypt/live/tu-dominio.com/privkey.pem
```

6. Reinicia el servicio:

```bash
sudo systemctl restart wazuh-dashboard
```

## 3.4 Solución de Problemas Comunes de Conectividad

### Problema: No Puedo Acceder al Dashboard Remotamente

**Posibles causas y soluciones:**

1. **El firewall está bloqueando el acceso:**
   
   Verifica la configuración del firewall:
   
   ```bash
   sudo ufw status
   ```
   
   Si el puerto 443 no está permitido, habilítalo:
   
   ```bash
   sudo ufw allow 443/tcp
   ```

2. **El servicio Wazuh dashboard no está escuchando en todas las interfaces:**
   
   Verifica la configuración:
   
   ```bash
   sudo cat /etc/wazuh-dashboard/opensearch_dashboards.yml | grep server.host
   ```
   
   Si no está configurado como `0.0.0.0`, edita el archivo y cambia el valor.

3. **Problemas con los certificados SSL:**
   
   Verifica que los certificados existen y tienen los permisos correctos:
   
   ```bash
   ls -la /etc/wazuh-dashboard/certs/
   ```
   
   Si hay problemas con los permisos, corrige:
   
   ```bash
   sudo chmod 400 /etc/wazuh-dashboard/certs/*
   sudo chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs
   ```

### Problema: Errores de Certificado en el Navegador

**Posibles causas y soluciones:**

1. **Certificado autofirmado:**
   
   Los navegadores modernos advierten sobre certificados autofirmados. Puedes:
   
   - Aceptar temporalmente el riesgo en tu navegador
   - Instalar un certificado válido como se describió anteriormente
   - Añadir una excepción permanente para este certificado en tu navegador

2. **Fecha y hora incorrectas:**
   
   Asegúrate de que la fecha y hora del servidor y tu máquina local sean correctas:
   
   ```bash
   sudo apt install -y ntp
   sudo systemctl start ntp
   sudo systemctl enable ntp
   ```

### Problema: El Túnel SSH se Desconecta Frecuentemente

**Posibles causas y soluciones:**

1. **Problemas de red intermitentes:**
   
   Utiliza AutoSSH como se describió anteriormente para mantener conexiones persistentes.

2. **Configuración SSH restrictiva en el servidor:**
   
   Edita `/etc/ssh/sshd_config` en el servidor:
   
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
   
   Asegúrate de que las siguientes opciones estén configuradas:
   
   ```
   TCPKeepAlive yes
   ClientAliveInterval 60
   ClientAliveCountMax 3
   ```
   
   Reinicia el servicio SSH:
   
   ```bash
   sudo systemctl restart sshd
   ```

### Problema: Rendimiento Lento a Través del Túnel SSH

**Posibles causas y soluciones:**

1. **Compresión no habilitada:**
   
   Utiliza la opción de compresión al crear el túnel:
   
   ```bash
   ssh -C -L 8443:localhost:443 usuario@ip_servidor_remoto
   ```

2. **Ancho de banda limitado:**
   
   Considera utilizar una conexión VPN en lugar de un túnel SSH para un mejor rendimiento.

Con estas configuraciones y soluciones, deberías poder acceder de manera segura y eficiente al Wazuh dashboard desde ubicaciones remotas. En la siguiente sección, abordaremos la implementación de Docker en el servidor para preparar el entorno para el monitoreo de contenedores.
