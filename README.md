# proyecto-SIS313
Es el repositorio del proyecto de SIS313

================================================================================
🚀  PROYECTO FINAL SIS313: SISTEMA DE SALAS DE ESPERA Y FILAS VIRTUALES
================================================================================

> Asignatura: SIS313: Infraestructura, Plataformas Tecnológicas y Redes
> Semestre:   2/2025
> Docente:    Ing. Marcelo Quispe Ortega

================================================================================
👥  MIEMBROS DEL EQUIPO (GRUPO VIRTUAL QUEUE)
================================================================================

1. Duran Chambi Benjamin Ricardo
   - Rol: Arquitecto de Backend & Proxy (VM-PROXY y VM-APP).
   - Contacto: Ricardo

2. Escobar Moscoso Jorge Gabriel
   - Rol: Administrador de Datos (VM-REDIS, Persistencia).
   - Contacto: GitHub @jogaesmo

3. Onofre Alanoca Roy
   - Rol: Ingeniero de Observabilidad (VM-MON).
   - Contacto: GitHub @RoyOnofre

================================================================================
🎯  I. OBJETIVO DEL PROYECTO
================================================================================

Diseñar e implementar una arquitectura de "Cola Virtual" (Virtual Waiting Room)
distribuida para el sistema universitario SUNiver.

El objetivo es interceptar el 100% del tráfico entrante, aplicar Rate Limiting
para limitar la concurrencia a un umbral seguro (ej. 5 usuarios activos) y
redirigir el exceso de tráfico a una sala de espera estática HTML. Esto
garantiza la disponibilidad del servicio crítico bajo ataques o saturación.

================================================================================
🌐  IV. DISEÑO DE LA INFRAESTRUCTURA (TOPOLOGÍA)
================================================================================

El sistema se divide en 4 Máquinas Virtuales (VMs) conectadas en red local:

1. VM-PROXY (Gateway):  Nginx. Recibe todo el tráfico y filtra.
   Hostname: proxy-server.local
   
2. VM-APP (Lógica):     Node.js. Decide si muestras Login o Sala de Espera.
   Hostname: app-server.local
   
3. VM-REDIS (Estado):   Redis Server. Guarda el contador de personas en RAM.
   Hostname: redis-server.local
   
4. VM-MON (Monitor):    Prometheus + Grafana. Vigila el sistema.
   Hostname: monitor-server.local

================================================================================
📋  V. GUÍA DE IMPLEMENTACIÓN DETALLADA (PASO A PASO)
================================================================================
Sigue estos pasos en orden estricto para levantar la infraestructura.

--------------------------------------------------------------------------------
FASE 0: PRE-REQUISITOS DE RED (EN TODAS LAS 4 VMs)
--------------------------------------------------------------------------------
Objetivo: Que las máquinas se encuentren por nombre (.local) sin usar IPs fijas.

1. Actualiza e instala las herramientas de red y el servicio Avahi (mDNS):
   $ sudo apt update
   $ sudo apt install -y net-tools curl avahi-daemon libnss-mdns

2. Verifica la conectividad (Ejemplo desde VM-PROXY hacia VM-APP):
   $ ping app-server.local
   (Si responde, el sistema de nombres funciona correctamente).

--------------------------------------------------------------------------------
FASE 1: CONFIGURACIÓN DE DATOS (EN VM-REDIS)
--------------------------------------------------------------------------------
Hostname requerido: redis-server.local

1. Instalar Redis Server:
   $ sudo apt install redis-server -y

2. Configurar para permitir conexiones externas (CRÍTICO):
   Por defecto, Redis solo escucha al propio servidor. Debemos abrirlo.
   
   $ sudo nano /etc/redis/redis.conf
   
   BUSCA la línea:
   bind 127.0.0.1
   
   CÁMBIALA por:
   bind 0.0.0.0

3. Reiniciar el servicio para aplicar cambios:
   $ sudo systemctl restart redis-server

4. (Opcional) Limpiar la base de datos para empezar de cero:
   $ redis-cli FLUSHALL

--------------------------------------------------------------------------------
FASE 2: CONFIGURACIÓN DE LÓGICA (EN VM-APP)
--------------------------------------------------------------------------------
Hostname requerido: app-server.local

1. Instalar entorno Node.js:
   $ sudo apt install nodejs npm -y

2. Crear carpeta del proyecto e instalar dependencias:
   $ mkdir proyecto-cola && cd proyecto-cola
   $ npm init -y
   $ npm install express redis

3. Crear el código del "Portero" (index.js):
   $ nano index.js

   COPIA Y PEGA EL SIGUIENTE CÓDIGO (Lógica del proyecto):
   -------------------------------------------------------
   const express = require('express');
   const redis = require('redis');
   const app = express();
   const PORT = 3000;
   const MAX_CAPACITY = 5; // Aforo máximo permitido
   const REDIS_KEY = 'suniver:active_users';

   // 1. Conexión a VM-REDIS
   const client = redis.createClient({
       socket: {
           host: 'redis-server.local', // Hostname de la Fase 1
           port: 6379
       }
   });
   client.connect();
   client.on('error', err => console.error('Redis Error:', err));

   // 2. Lógica de Admisión
   app.get('/', async (req, res) => {
       try {
           // Incrementamos contador atómico en Redis
           const activeUsers = await client.incr(REDIS_KEY);

           if (activeUsers <= MAX_CAPACITY) {
               // ESCENARIO A: HAY CUPO (PANTALLA VERDE - LOGIN)
               // Simulamos que el usuario está 10 seg y se va
               setTimeout(() => client.decr(REDIS_KEY), 10000);
               
               res.send(`
                 <body style="background-color: #2ecc71; color: white; text-align: center; font-family: sans-serif;">
                   <h1>✅ BIENVENIDO AL SISTEMA</h1>
                   <p>Usuarios: ${activeUsers} / ${MAX_CAPACITY}</p>
                 </body>
               `);
           } else {
               // ESCENARIO B: NO HAY CUPO (PANTALLA NARANJA - ESPERA)
               // Restamos al usuario porque no entró
               await client.decr(REDIS_KEY);
               
               res.send(`
                 <body style="background-color: #e67e22; color: white; text-align: center; font-family: sans-serif;">
                   <h1>🟠 SALA DE ESPERA</h1>
                   <p>Sistema saturado. Recargando...</p>
                   <meta http-equiv="refresh" content="5">
                 </body>
               `);
           }
       } catch (error) {
           res.status(503).send('<h1>🔴 ERROR DE SERVICIO</h1>');
       }
   });

   app.listen(PORT, () => console.log(`App en puerto ${PORT}`));
   -------------------------------------------------------

4. Ejecutar la aplicación en segundo plano:
   $ nohup node index.js &

--------------------------------------------------------------------------------
FASE 3: CONFIGURACIÓN DE ACCESO Y RATE LIMIT (EN VM-PROXY)
--------------------------------------------------------------------------------
Hostname requerido: proxy-server.local

1. Instalar Nginx:
   $ sudo apt install nginx -y

2. Configurar el Rate Limiting (Paso A):
   Editar configuración principal.
   $ sudo nano /etc/nginx/nginx.conf
   
   DENTRO del bloque "http { ... }", añade esta línea:
   limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;

3. Configurar el Proxy Inverso y Upstream (Paso B):
   Editar el sitio por defecto.
   $ sudo nano /etc/nginx/sites-available/default
   
   BORRA TODO y pega esta configuración optimizada:
   -------------------------------------------------------
   # Definimos dónde está la aplicación (VM-APP)
   upstream backend_app {
       server app-server.local:3000;
   }

   server {
       listen 80 default_server;
       listen [::]:80 default_server;
       server_name _;

       location / {
           # Aplicar el Rate Limit definido antes
           limit_req zone=one burst=5 nodelay;

           # Enviar tráfico a VM-APP
           proxy_pass http://backend_app;
           
           # Cabeceras para pasar la IP real
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
       }
   }
   -------------------------------------------------------

4. Verificar y Reiniciar Nginx:
   $ sudo nginx -t
   $ sudo systemctl restart nginx

--------------------------------------------------------------------------------
FASE 4: MONITOREO (EN VM-MON)
--------------------------------------------------------------------------------
Hostname requerido: monitor-server.local

1. Editar configuración de Prometheus para "leer" a VM-APP:
   $ sudo nano /etc/prometheus/prometheus.yml

2. Añadir bajo "scrape_configs":
   - job_name: 'node_app'
     scrape_interval: 5s
     static_configs:
       - targets: ['app-server.local:3000']

3. Reiniciar Prometheus:
   $ sudo systemctl restart prometheus

4. Entrar a Grafana (http://monitor-server.local:3000) y ver métricas.

================================================================================
⚠️  VI. CÓMO PROBAR QUE FUNCIONA
================================================================================

1. PRUEBA DE ACCESO NORMAL:
   - Abre el navegador y ve a: http://proxy-server.local
   - Deberías ver la pantalla VERDE (Bienvenido) si hay menos de 5 usuarios.

2. PRUEBA DE SATURACIÓN (SALA DE ESPERA):
   - Abre 6 pestañas diferentes o usa distintos dispositivos.
   - A partir de la 6ta conexión simultánea, verás la pantalla NARANJA.
   - Espera 10 segundos (cuando un usuario "sale"), la pantalla naranja se
     recargará sola y cambiará a verde.

3. PRUEBA DE FALLO (CIRCUIT BREAKER):
   - Apaga la VM-APP.
   - Nginx debería devolver un error 502 Bad Gateway, pero si configuras
     una página de error personalizada en Nginx, podrías mantener el mensaje.

================================================================================
📚  VII. CONCLUSIONES Y TECNOLOGÍAS USADAS
================================================================================

- Redis en RAM eliminó la latencia de base de datos SQL.
- Nginx protegió la aplicación de ataques DDoS capa 7.
- mDNS permitió desplegar sin configurar IPs estáticas.

FIN DEL DOCUMENTO
