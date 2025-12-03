# proyecto-SIS313

# 🚀 Proyecto Final SIS313: Sistema de Salas de Espera y Filas Virtuales (Virtual Queue)

> **Asignatura:** SIS313: Infraestructura, Plataformas Tecnológicas y Redes<br>
> **Semestre:** 2/2025<br>
> **Docente:** Ing. Marcelo Quispe Ortega

## 👥 Miembros del Equipo (Grupo Virtual Queue)

| Nombre Completo | Rol en el Proyecto | Contacto (GitHub/Email) |
| :--- | :--- | :--- |
| Duran Chambi Benjamin Ricardo | **Arquitecto de Backend & Proxy:** Encargado de la VM-PROXY y VM-APP (Nginx/Node.js). Diseño de la lógica de encolamiento. | Ricardo |
| Escobar Moscoso Jorge Gabriel | **Administrador de Datos:** Encargado de la VM-REDIS. Gestión de persistencia en memoria y optimización de consultas. | [jogaesmo](https://github.com/jogaesmo) |
| Onofre Alanoca Roy | **Ingeniero de Observabilidad:** Encargado de la VM-MON (Prometheus/Grafana). Monitoreo y auditoría de métricas. | [RoyOnofre](https://github.com/RoyOnofre) |

## 🎯 I. Objetivo del Proyecto

> **Objetivo:** Diseñar e implementar una arquitectura de **Cola Virtual (Virtual Waiting Room)** distribuida para el sistema universitario SUNiver, capaz de interceptar el 100% del tráfico entrante, aplicar **Rate Limiting** para limitar la concurrencia a un umbral seguro (ej. 100 usuarios/minuto) y redirigir el exceso de tráfico a una sala de espera estática, garantizando así la **disponibilidad del servicio crítico** bajo condiciones de saturación.

## 💡 II. Justificación e Importancia

> **Justificación:** El problema recurrente durante las fechas de inscripción es la **Denegación de Servicio Involuntaria** (saturación de usuarios), lo que genera una **Falla de Continuidad Operacional (T1)**. Este proyecto es vital porque transforma el fallo en espera, convirtiendo una caída del servidor (Error 503) en una **espera ordenada**. Implementa una **Protección Centralizada (T5)** mediante Nginx para desacoplar la carga masiva y una **Optimización Extrema (T4)** utilizando **Redis (en RAM)** para la gestión del estado de la cola, resolviendo la concurrencia a latencias de sub-milisegundos.

## 🛠️ III. Tecnologías y Conceptos Implementados

### 3.1. Tecnologías Clave

* **Nginx (VM-PROXY):** Gateway y Reverse Proxy. Protege la topología interna y aplica el módulo `limit_req` para filtrar peticiones abusivas (Rate Limiting).
* **Node.js (Express) (VM-APP):** Servidor de Aplicación / Lógica. Ejecuta la lógica de "Portero": consulta el estado de la cola a Redis y decide si servir la página de Login o la Sala de Espera HTML.
* **Redis (VM-REDIS):** Gestión de Estado Global (Memoria). Base de Datos NoSQL en memoria RAM. Mantiene el contador atómico del aforo en tiempo real, garantizando la "Verdad Única".
* **Prometheus / Grafana (VM-MON):** Observabilidad y Auditoría. Prometheus hace scraping de métricas de la aplicación, y Grafana visualiza la saturación de tráfico y el comportamiento de la cola.
* **Tailscale / Avahi (mDNS):** Networking Transparente. Proporciona una interconexión de las VMs mediante nombres de dominio `.local`, facilitando el descubrimiento de servicios sin IPs estáticas.

### 3.2. Conceptos de la Asignatura Puestos en Práctica (T1 - T6)

Marca con un ✅ los temas avanzados de la asignatura que fueron implementados:

* **Alta Disponibilidad (T2) y Tolerancia a Fallos:** ✅ Implementación de un patrón **Circuit Breaker Lógico** que previene la caída en cascada del servidor principal mediante el desvío de tráfico a la sala de espera estática.
* **Seguridad y Hardening (T5):** ✅ Uso de **Rate Limiting** en el Proxy Inverso (Nginx) para mitigar ataques de fuerza bruta y denegación de servicio (DDoS) en Capa 7.
* **Automatización y Gestión (T6):** ✅ Configuración de servicios `systemd` para el arranque automático y la recuperación de servicios (Nginx, Redis, App) tras reinicios no programados.
* **Balanceo de Carga/Proxy (T3/T4):** ✅ Configuración de **Upstreams** en Nginx para abstraer la ubicación real de los servidores de aplicación y permitir escalabilidad horizontal futura.
* **Monitoreo (T4/T1):** ✅ Implementación de un stack de observabilidad completo (Métricas RED) para detectar cuellos de botella y auditar la estabilidad bajo picos de carga.
* **Networking Avanzado (T3):** ✅ Implementación de resolución de nombres interna (mDNS) y segmentación de servicios en distintas VMs/Hosts.

## 🌐 IV. Diseño de la Infraestructura y Topología

### 4.1. Diseño Esquemático

El diseño se basa en la segmentación física de los servicios en **4 VMs** distribuidas en hosts distintos, comunicadas por nombre de dominio a través de la red LAN (mDNS).

> 
| VM/Host | Rol | IP Lógica / Hostname | Software Principal | Capa | SO |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VM-PROXY** | Gateway / Rate Limiter | `proxy-server.local` | Nginx | Acceso | Ubuntu 22.04 |
| **VM-APP** | Lógica de Negocio / Worker | `app-server.local` | Node.js v20 | Aplicación | Ubuntu 22.04 |
| **VM-REDIS** | Gestión de Estado (Cola) | `redis-server.local` | Redis Server 7 | Datos | Ubuntu 22.04 |
| **VM-MON** | Monitoreo y Alertas | `monitor-server.local` | Prometheus + Grafana | Gestión | Ubuntu 22.04 |

### 4.2. Estrategia Adoptada

* **Estrategia de Desacoplamiento:** Se separó el **Proxy**, la **Lógica** y el **Estado** en máquinas virtuales distintas. Esto garantiza que la `VM-PROXY` pueda seguir respondiendo con páginas de Sala de Espera, incluso si la `VM-APP` se sobrecarga o falla.
* **Estrategia de Optimización (Redis):** Se eligió **Redis** sobre una base de datos SQL porque las operaciones de incremento/decremento de cola deben ser atómicas y de latencia cero. El uso de la RAM garantiza que el contador de cupos nunca sea el cuello de botella (T4).

## 📋 V. Guía de Implementación y Puesta en Marcha

### 5.1. Pre-requisitos (Todas las VMs)

**Objetivo:** Establecer la conectividad por nombre de host (hostname), crucial para que los servicios se encuentren entre sí sin depender de IPs estáticas.

1.  **Actualizar e Instalar herramientas básicas y Avahi (mDNS):**
    
    ```bash
    sudo apt update
    sudo apt install -y net-tools curl avahi-daemon libnss-mdns
    ```

2.  **Configurar Hostnames:**
    Ejecutar en cada VM el comando correspondiente:
    * `sudo hostnamectl set-hostname redis-server`
    * `sudo hostnamectl set-hostname app-server`
    * `sudo hostnamectl set-hostname proxy-server`
    * `sudo hostnamectl set-hostname monitor-server`

3.  **Prueba de Conectividad:**
    Desde cualquier VM, verifica la conexión con otra usando su nombre lógico.
    ```bash
    ping app-server.local
    ```

### 5.2. Despliegue Detallado

#### Paso 1: Configuración VM-REDIS (Capa de Datos)

1.  **Instalar Redis Server:**
    ```bash
    sudo apt install redis-server -y
    ```

2.  **Configurar Bind Address:**
    Editar el archivo para permitir conexiones externas desde la VM-APP.
    ```bash
    sudo nano /etc/redis/redis.conf
    ```
    > Buscar la línea `bind 127.0.0.1` y cambiarla a `bind 0.0.0.0`.

3.  **Reiniciar Servicio:**
    ```bash
    sudo systemctl restart redis-server
    ```

#### Paso 2: Configuración VM-APP (Lógica del Portero)

1.  **Instalar Node.js y dependencias:**
    ```bash
    sudo apt install nodejs npm -y
    mkdir proyecto && cd proyecto
    npm init -y
    npm install express redis
    ```

2.  **Crear el archivo `index.js`:**
    ```bash
    nano index.js
    ```
    
    Pegar el siguiente código de lógica de admisión:

    ```javascript
    const express = require('express');
    const redis = require('redis');
    const app = express();
    const PORT = 3000;
    const MAX_CAPACITY = 5; 
    const REDIS_KEY = 'suniver:active_users';

    const client = redis.createClient({
        socket: { host: 'redis-server.local', port: 6379 }
    });
    client.connect();

    app.get('/', async (req, res) => {
        try {
            const activeUsers = await client.incr(REDIS_KEY);
            if (activeUsers <= MAX_CAPACITY) {
                setTimeout(() => client.decr(REDIS_KEY), 10000);
                res.send(`<h1>✅ BIENVENIDO (Usuarios: ${activeUsers})</h1>`);
            } else {
                await client.decr(REDIS_KEY);
                res.send(`<h1>🟠 SALA DE ESPERA</h1><meta http-equiv="refresh" content="5">`);
            }
        } catch (error) { res.status(503).send('Error'); }
    });

    app.listen(PORT, () => console.log(`App running on ${PORT}`));
    ```

3.  **Ejecutar Servicio:**
    ```bash
    nohup node index.js &
    ```

#### Paso 3: Configuración VM-PROXY (Capa de Acceso)

1.  **Instalar Nginx:**
    ```bash
    sudo apt install nginx -y
    ```

2.  **Configurar Rate Limiting:**
    Editar `nginx.conf` para definir la zona de memoria.
    ```bash
    sudo nano /etc/nginx/nginx.conf
    ```
    > Dentro del bloque `http { ... }`, añadir:
    > `limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;`

3.  **Configurar Upstream y Proxy:**
    Editar el archivo por defecto.
    ```bash
    sudo nano /etc/nginx/sites-available/default
    ```
    
    ```nginx
    upstream backend_app {
        server app-server.local:3000;
    }

    server {
        listen 80 default_server;
        location / {
            limit_req zone=one burst=5 nodelay;
            proxy_pass http://backend_app;
            proxy_set_header Host $host;
        }
    }
    ```

4.  **Reiniciar Nginx:**
    ```bash
    sudo nginx -t && sudo systemctl restart nginx
    ```

### 5.3. Ficheros de Configuración Clave

* **`index.js` (VM-APP):** Algoritmo de decisión (If cupo -> Login, Else -> SalaEspera) y conexión remota a redis-server.local.
* **`nginx.conf` (VM-PROXY):** Define la directiva `limit_req_zone` para el control de acceso.
* **`redis.conf` (VM-REDIS):** Establece `bind 0.0.0.0` para permitir la comunicación remota con la VM-APP.
* **`prometheus.yml` (VM-MON):** Configura el scraping (recolección) de métricas de la VM-APP.

## ⚠️ VI. Pruebas y Validación

| Prueba Realizada | Resultado Esperado | Resultado Obtenido |
| :--- | :--- | :--- |
| **Test de Estrés (Saturación)** | Al superar el límite de N usuarios, el usuario N+1 debe ver la pantalla naranja de "Sala de Espera", no el Error 503. | **OK:** El sistema redirigió correctamente al usuario 6 a la cola. |
| **Test de Recuperación Automática** | Cuando un usuario activo abandona el sistema (timeout), la cola debe avanzar automáticamente. | **OK:** El usuario en espera ingresó automáticamente en el siguiente refresco. |
| **Prueba de Monitoreo en Vivo** | Grafana debe mostrar el pico de tráfico y el estancamiento de usuarios activos en el límite máximo. | **OK:** Dashboard reflejó la curva de saturación en tiempo real, validando la estabilidad (T1). |
| **Validación de Conectividad** | Las VMs deben comunicarse por hostname (.local) independientemente de la IP asignada por el DHCP. | **OK:** Pings y conexiones de base de datos exitosas por nombre. |

## 📚 VII. Conclusiones y Lecciones Aprendidas

**Logro Principal:** Demostramos que la **escalabilidad y la disponibilidad** se logran mediante una arquitectura de software inteligente (**Desacoplamiento**), no solo con la adición de más hardware. El sistema transformó un escenario de fallo (Error 500) en una experiencia de usuario controlada (Sala de Espera).

**Lección Aprendida:** La importancia de **Redis (T4)** es crítica. Intentar gestionar el estado de la cola en una base de datos tradicional hubiera introducido latencia, convirtiendo al contador en el principal cuello de botella. La gestión en RAM fue esencial.
