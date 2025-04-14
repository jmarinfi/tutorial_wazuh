# 1. Infraestructura y Requisitos Previos

Antes de comenzar con la instalación de Wazuh, es fundamental asegurarse de que contamos con la infraestructura adecuada y cumplimos con todos los requisitos previos. Esta sección detalla los aspectos clave a considerar para garantizar una implementación exitosa.

## 1.1 Requisitos Hardware para el Servidor Virtual

Wazuh puede implementarse en un único servidor o en una configuración distribuida con múltiples nodos. Los requisitos de hardware varían según el tamaño de la implementación y la cantidad de agentes que se monitorizarán. A continuación, se presentan las recomendaciones para diferentes escenarios:

### Implementación de un solo nodo (hasta 50 agentes)

| Componente | Mínimo | Recomendado |
|------------|--------|-------------|
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Almacenamiento | 50 GB | 100 GB |

### Implementación multi-nodo (más de 50 agentes)

Para entornos más grandes, se recomienda una arquitectura distribuida:

**Nodo Wazuh Manager:**
| Componente | Mínimo | Recomendado |
|------------|--------|-------------|
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Almacenamiento | 50 GB | 100 GB |

**Nodo Wazuh Indexer:**
| Componente | Mínimo | Recomendado |
|------------|--------|-------------|
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Almacenamiento | 50 GB por millón de eventos (90 días) | 100 GB por millón de eventos (90 días) |

**Nodo Wazuh Dashboard:**
| Componente | Mínimo | Recomendado |
|------------|--------|-------------|
| CPU | 2 cores | 4 cores |
| RAM | 4 GB | 8 GB |
| Almacenamiento | 10 GB | 20 GB |

### Consideraciones de almacenamiento

El espacio de almacenamiento necesario depende principalmente de la cantidad de eventos generados por segundo (EPS) y el período de retención deseado. Según la documentación oficial de Wazuh, se puede estimar el almacenamiento necesario de la siguiente manera:

| Tipo de endpoint | EPS promedio | Almacenamiento en Wazuh Server (GB/90 días) |
|------------------|--------------|-------------------------------------------|
| Servidores | 0.25 | 0.1 |
| Estaciones de trabajo | 0.1 | 0.04 |
| Dispositivos de red | 0.5 | 0.2 |

Por ejemplo, para un entorno con 80 estaciones de trabajo, 10 servidores y 10 dispositivos de red, el almacenamiento necesario en el servidor Wazuh para 90 días de alertas sería aproximadamente 6 GB.

## 1.2 Versiones de Ubuntu Server

Wazuh es compatible con varias distribuciones de Linux, pero para este tutorial nos centraremos en Ubuntu Server, que es una de las plataformas más populares y bien soportadas. A continuación, se presenta una comparativa de las versiones compatibles:

### Ubuntu 18.04 LTS (Bionic Beaver)
- **Soporte hasta:** Abril 2023 (soporte estándar) / Abril 2028 (soporte extendido)
- **Kernel:** 4.15
- **Ventajas:** Estable y ampliamente probado
- **Desventajas:** Paquetes más antiguos, cerca del fin de su soporte estándar

### Ubuntu 20.04 LTS (Focal Fossa)
- **Soporte hasta:** Abril 2025 (soporte estándar) / Abril 2030 (soporte extendido)
- **Kernel:** 5.4
- **Ventajas:** Buen equilibrio entre estabilidad y actualización, soporte a largo plazo
- **Desventajas:** No incluye las características más recientes de Ubuntu

### Ubuntu 22.04 LTS (Jammy Jellyfish)
- **Soporte hasta:** Abril 2027 (soporte estándar) / Abril 2032 (soporte extendido)
- **Kernel:** 5.15
- **Ventajas:** Versión más reciente con soporte a largo plazo, paquetes actualizados, mejor soporte para hardware moderno
- **Desventajas:** Podría tener algunos problemas iniciales de compatibilidad con software específico

### Ubuntu 24.04 LTS (Noble Numbat)
- **Soporte hasta:** Abril 2029 (soporte estándar) / Abril 2034 (soporte extendido)
- **Kernel:** 6.8
- **Ventajas:** La versión más reciente con las últimas características y mejoras
- **Desventajas:** Al ser muy reciente, podría tener problemas de compatibilidad con algunas aplicaciones

### Recomendación

Para una implementación de producción de Wazuh, se recomienda **Ubuntu 22.04 LTS** por las siguientes razones:
- Ofrece un buen equilibrio entre estabilidad y actualización
- Tiene soporte a largo plazo hasta 2027 (estándar)
- Es completamente compatible con la versión más reciente de Wazuh
- Incluye mejoras significativas en seguridad y rendimiento respecto a versiones anteriores

## 1.3 Puertos Necesarios para la Comunicación

Para que Wazuh funcione correctamente, es necesario configurar adecuadamente los puertos de comunicación entre sus componentes. A continuación, se detallan los puertos que deben estar abiertos:

### Puertos para el Wazuh Manager

| Puerto | Protocolo | Descripción |
|--------|-----------|-------------|
| 1514 | TCP/UDP | Comunicación entre agentes y manager (conexión segura) |
| 1515 | TCP | Servicio de registro de agentes |
| 1516 | TCP | Clúster de Wazuh (comunicación entre nodos) |
| 55000 | TCP | API REST de Wazuh |

### Puertos para el Wazuh Indexer

| Puerto | Protocolo | Descripción |
|--------|-----------|-------------|
| 9200 | TCP | API REST del Wazuh Indexer (HTTP) |
| 9300-9400 | TCP | Comunicación entre nodos del clúster de Wazuh Indexer |

### Puertos para el Wazuh Dashboard

| Puerto | Protocolo | Descripción |
|--------|-----------|-------------|
| 443 | TCP | Interfaz web HTTPS |

### Puertos para Monitoreo Agentless

| Puerto | Protocolo | Descripción |
|--------|-----------|-------------|
| 22 | TCP | SSH para monitoreo agentless |

### Configuración de Firewall

A continuación, se muestra cómo configurar estos puertos utilizando UFW (Uncomplicated Firewall) en Ubuntu:

```bash
# Habilitar UFW
sudo ufw enable

# Configurar puertos para Wazuh Manager
sudo ufw allow 1514/tcp
sudo ufw allow 1514/udp
sudo ufw allow 1515/tcp
sudo ufw allow 1516/tcp
sudo ufw allow 55000/tcp

# Configurar puertos para Wazuh Indexer
sudo ufw allow 9200/tcp
sudo ufw allow 9300:9400/tcp

# Configurar puertos para Wazuh Dashboard
sudo ufw allow 443/tcp

# Configurar puertos para Monitoreo Agentless
sudo ufw allow 22/tcp

# Verificar la configuración
sudo ufw status verbose
```

> **Nota**: Si estás utilizando una configuración distribuida, asegúrate de que estos puertos estén abiertos entre los diferentes nodos de tu infraestructura.

## 1.4 Configuración SSH desde un PC Ubuntu hacia el Servidor Remoto

Para administrar eficientemente tu servidor Wazuh, es recomendable configurar el acceso SSH desde tu PC local hacia el servidor remoto. Esto facilitará la ejecución de comandos y la transferencia de archivos.

### Generación de Claves SSH

1. En tu PC local con Ubuntu, genera un par de claves SSH si aún no las tienes:

```bash
ssh-keygen -t rsa -b 4096
```

2. Durante el proceso, se te pedirá que especifiques la ubicación para guardar las claves (por defecto, `~/.ssh/id_rsa`) y una frase de contraseña opcional (recomendada para mayor seguridad).

### Transferencia de la Clave Pública al Servidor Remoto

Existen dos métodos para transferir tu clave pública al servidor:

#### Método 1: Usando ssh-copy-id (recomendado)

```bash
ssh-copy-id usuario@ip_servidor_remoto
```

Por ejemplo:
```bash
ssh-copy-id ubuntu@192.168.1.100
```

Se te pedirá la contraseña del usuario en el servidor remoto.

#### Método 2: Transferencia manual

Si `ssh-copy-id` no está disponible, puedes transferir la clave manualmente:

```bash
cat ~/.ssh/id_rsa.pub | ssh usuario@ip_servidor_remoto "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### Configuración del Archivo SSH Config

Para simplificar las conexiones SSH, puedes configurar el archivo `~/.ssh/config` en tu PC local:

1. Crea o edita el archivo:

```bash
nano ~/.ssh/config
```

2. Añade una configuración para tu servidor Wazuh:

```
Host wazuh-server
    HostName 192.168.1.100
    User ubuntu
    Port 22
    IdentityFile ~/.ssh/id_rsa
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

3. Guarda el archivo (Ctrl+O, luego Enter) y sal del editor (Ctrl+X).

Con esta configuración, podrás conectarte simplemente usando:

```bash
ssh wazuh-server
```

### Verificación de la Conexión SSH

Para verificar que la configuración SSH funciona correctamente:

```bash
ssh wazuh-server
```

Si has configurado correctamente las claves SSH, deberías poder acceder sin introducir una contraseña (aunque es posible que se te pida la frase de contraseña si la configuraste al generar las claves).

### Consideraciones de Seguridad para SSH

Para mejorar la seguridad de tus conexiones SSH, considera implementar las siguientes medidas:

1. **Deshabilitar el acceso por contraseña**:

   Edita el archivo `/etc/ssh/sshd_config` en el servidor remoto:
   
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
   
   Busca y modifica las siguientes líneas:
   
   ```
   PasswordAuthentication no
   ChallengeResponseAuthentication no
   UsePAM no
   ```
   
   Reinicia el servicio SSH:
   
   ```bash
   sudo systemctl restart sshd
   ```

2. **Cambiar el puerto SSH por defecto**:

   En el mismo archivo `/etc/ssh/sshd_config`, cambia la línea:
   
   ```
   Port 22
   ```
   
   a un puerto no estándar, por ejemplo:
   
   ```
   Port 2222
   ```
   
   No olvides actualizar la configuración de tu firewall y el archivo `~/.ssh/config` en tu PC local.

3. **Limitar los intentos de acceso**:

   Añade las siguientes líneas a `/etc/ssh/sshd_config`:
   
   ```
   MaxAuthTries 3
   MaxSessions 5
   ```

4. **Implementar Fail2Ban**:

   Fail2Ban es una herramienta que ayuda a prevenir ataques de fuerza bruta:
   
   ```bash
   sudo apt update
   sudo apt install fail2ban
   sudo systemctl enable fail2ban
   sudo systemctl start fail2ban
   ```
   
   Configura Fail2Ban para SSH editando `/etc/fail2ban/jail.local`:
   
   ```bash
   sudo nano /etc/fail2ban/jail.local
   ```
   
   Añade:
   
   ```
   [sshd]
   enabled = true
   port = ssh
   filter = sshd
   logpath = /var/log/auth.log
   maxretry = 3
   bantime = 3600
   ```

Con estos requisitos previos configurados correctamente, estamos listos para proceder con la instalación del stack de servidor Wazuh en la siguiente sección.
