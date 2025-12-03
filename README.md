# proyecto-SIS313

# 🚀 Proyecto Final SIS313: Sistema de Salas de Espera y Filas Virtuales (Virtual Queue)

> **Asignatura:** SIS313: Infraestructura, Plataformas Tecnológicas y Redes<br>
> **Semestre:** 2/2025<br>
> **Docente:** Ing. Marcelo Quispe Ortega

## 👥 Miembros del Equipo (Grupo Virtual Queue)

| Nombre Completo | Rol en el Proyecto | Contacto (GitHub/Email) |
| :--- | :--- | :--- |
| **Duran Chambi Benjamin Ricardo** | Arquitecto de Backend & Proxy (Nginx/Node.js) | Ricardo |
| **Escobar Moscoso Jorge Gabriel** | Administrador de Datos (Redis/Persistencia) | [jogaesmo](https://github.com/jogaesmo) |
| **Onofre Alanoca Roy** | Ingeniero de Observabilidad (Prometheus/Grafana) | [RoyOnofre](https://github.com/RoyOnofre) |

## 🎯 I. Objetivo del Proyecto

> **Objetivo:** Diseñar e implementar una arquitectura de **Cola Virtual (Virtual Waiting Room)** distribuida para el sistema universitario SUNiver, capaz de interceptar el 100% del tráfico entrante, aplicar **Rate Limiting** para limitar la concurrencia a un umbral seguro (ej. 100 usuarios/minuto) y redirigir el exceso de tráfico a una sala de espera estática, garantizando así la **disponibilidad del servicio crítico** bajo condiciones de saturación.

## 💡 II. Justificación e Importancia

> **Justificación:** El problema recurrente durante las fechas de inscripción es la **Denegación de Servicio Involuntaria** (saturación de usuarios), lo que genera una Falla de Continuidad Operacional (T1). Este proyecto transforma el fallo en espera, convirtiendo un Error 503 en una espera ordenada. Implementa **Protección Centralizada (T5)** mediante Nginx para desacoplar la carga y **Optimización Extrema (T4)** utilizando Redis en RAM para gestionar la concurrencia a latencias de sub-milisegundos, resolviendo los cuellos de botella de las bases de datos tradicionales.

## 🛠️ III. Tecnologías y Conceptos Implementados

### 3.1. Tecnologías Clave

* **Nginx (VM-PROXY):** [Función: Gateway y Reverse Proxy. Protege la topología interna y aplica el módulo `limit_req` para filtrar peticiones abusivas (Rate Limiting).]
* **Node.js (Express) (VM-APP):** [Función: Lógica de "Portero". Consulta el estado de la cola a Redis y decide si servir la página de Login o la Sala de Espera HTML.]
* **Redis (VM-REDIS):** [Función: Base de Datos NoSQL en memoria RAM. Mantiene el contador atómico del aforo en tiempo real, garantizando la "Verdad Única".]
* **Prometheus / Grafana (VM-MON):** [Función: Observabilidad. Prometheus hace scraping de métricas y Grafana visualiza la saturación de tráfico y el comportamiento de la cola.]
* **Tailscale / Avahi (mDNS):** [Función: Networking Transparente. Interconexión de las VMs mediante nombres de dominio `.local`, facilitando el descubrimiento de servicios sin IPs estáticas.]

### 3.2. Conceptos de la Asignatura Puestos en Práctica (T1 - T6)

Marca con un ✅ los temas avanzados de la asignatura que fueron implementados:

* **Alta Disponibilidad (T2) y Tolerancia a Fallos:** ✅ [Implementación de un patrón Circuit Breaker Lógico que previene la caída en cascada del servidor principal desviando tráfico a sala de espera.]
* **Seguridad y Hardening (T5):** ✅ [Uso de Rate Limiting en el Proxy Inverso (Nginx) para mitigar ataques de fuerza bruta y DDoS en Capa 7.]
* **Automatización y Gestión (T6):** ✅ [Configuración de servicios `systemd` para el arranque automático y la recuperación de servicios tras reinicios.]
* **Balanceo de Carga/Proxy (T3/T4):** ✅ [Configuración de Upstreams en Nginx para abstraer la ubicación real de los servidores de aplicación.]
* **Monitoreo (T4/T1):** ✅ [Implementación de un stack de observabilidad completo (Métricas RED) para detectar cuellos de botella.]
* **Networking Avanzado (T3):** ✅ [Implementación de resolución de nombres interna (mDNS) y segmentación de servicios en distintas VMs.]

## 🌐 IV. Diseño de la Infraestructura y Topología

### 4.1. Diseño Esquemático

El diseño se basa en la segmentación física de los servicios en 4 VMs distribuidas en hosts distintos.

> 
| VM/Host | Rol | Hostname (.local) | Software Principal | Capa | SO |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VM-PROXY** | Gateway / Rate Limiter | `proxy-server.local` | Nginx | Acceso | Ubuntu 22.04 |
| **VM-APP** | Lógica / Worker | `app-server.local` | Node.js v20 | Aplicación | Ubuntu 22.04 |
| **VM-REDIS** | Gestión de Estado | `redis-server.local` | Redis Server 7 | Datos | Ubuntu 22.04 |
| **VM-MON** | Monitoreo | `monitor-server.local` | Prometheus + Grafana | Gestión | Ubuntu 22.04 |

### 4.2. Estrategia Adoptada

* **Estrategia de Desacoplamiento:** Se separó el Proxy, la Lógica y el Estado. Esto garantiza que la VM-PROXY pueda seguir respondiendo con la Sala de Espera incluso si la VM-APP falla.
* **Estrategia de Optimización (Redis):** Se eligió Redis sobre SQL porque las operaciones de incremento/decremento deben ser atómicas y de latencia cero (RAM).

## 📋 V. Guía de Implementación y Puesta en Marcha

### 5.1. Pre-requisitos (Networking)
Ejecutar en **TODAS** las VMs para habilitar resolución por nombre:

```bash
sudo apt update
sudo apt install -y net-tools curl avahi-daemon libnss-mdns
# Verificar conectividad:
ping app-server.local

5.2. Despliegue Detallado (Paso a Paso)
Paso 1: Configurar Base de Datos (VM-REDIS)
Instalar Redis: sudo apt install redis-server -y

Permitir conexiones remotas editando el archivo redis.conf:

Bash

sudo nano /etc/redis/redis.conf
# CAMBIAR: bind 127.0.0.1  --> POR: bind 0.0.0.0

Reiniciar: sudo systemctl restart redis-serverPaso 2: Configurar Aplicación (VM-APP)Instalar Node.js: sudo apt install nodejs npm -yCrear proyecto:Bashmkdir proyecto && cd proyecto
npm init -y
npm install express redis
nano index.js
Código Fuente (index.js):JavaScriptconst express = require('express');
const redis = require('redis');
const app = express();
const client = redis.createClient({ socket: { host: 'redis-server.local', port: 6379 } });
client.connect();

app.get('/', async (req, res) => {
    const active = await client.incr('users');
    if (active <= 5) { // Aforo maximo 5
        setTimeout(() => client.decr('users'), 10000); // Simula 10s de uso
        res.send(`<h1>✅ BIENVENIDO (Usuarios: ${active})</h1>`);
    } else {
        await client.decr('users');
        res.send(`<h1>🟠 SALA DE ESPERA</h1><meta http-equiv="refresh" content="5">`);
    }
});
app.listen(3000);
Ejecutar: nohup node index.js &Paso 3: Configurar Proxy y Rate Limit (VM-PROXY)Instalar Nginx: sudo apt install nginx -yEditar nginx.conf para agregar el límite:Bashsudo nano /etc/nginx/nginx.conf
# Agregar en bloque http:
limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;
Configurar el sitio en /etc/nginx/sites-available/default:Nginxupstream backend { server app-server.local:3000; }
server {
    listen 80 default_server;
    location / {
        limit_req zone=one burst=5 nodelay;
        proxy_pass http://backend;
    }
}
Reiniciar: sudo systemctl restart nginx5.3. Ficheros de Configuración Clave/etc/redis/redis.conf: Configuración de Bind IP.index.js: Lógica de negocio y conexión a Redis./etc/nginx/nginx.conf: Definición de Zona de Rate Limit.⚠️ VI. Pruebas y ValidaciónPrueba RealizadaResultado EsperadoResultado ObtenidoTest de EstrésAl superar el límite de N usuarios, el usuario N+1 ve la "Sala de Espera".✅ OK: Redirección correcta.Test de RecuperaciónCuando un usuario sale, la cola avanza automáticamente.✅ OK: Ingreso automático.Monitoreo en VivoGrafana muestra el pico de tráfico y saturación.✅ OK: Métricas visibles.ConectividadLas VMs se comunican por hostname (.local).✅ OK: Pings exitosos.📚 VII. Conclusiones y Lecciones AprendidasSe logró implementar una arquitectura resiliente donde Redis en RAM fue la clave para eliminar la latencia en el conteo de usuarios. El desacoplamiento de servicios permitió que el Proxy siguiera funcionando (mostrando la sala de espera) incluso bajo estrés máximo de la aplicación.
