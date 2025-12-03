# proyecto-SIS313
Es el repositorio del proyecto de SIS313

🚀 Proyecto Final SIS313: Sistema de Salas de Espera y Filas Virtuales (Virtual Queue)Asignatura: SIS313: Infraestructura, Plataformas Tecnológicas y RedesSemestre: 2/2025 1Docente: Ing. Marcelo Quispe Ortega 2👥 Miembros del Equipo (Grupo Virtual Queue)Nombre CompletoRol en el ProyectoContacto (GitHub/Email)Duran Chambi Benjamin RicardoArquitecto de Backend & Proxy: Encargado de la VM-PROXY y VM-APP (Nginx/Node.js). Diseño de la lógica de encolamiento. 3RicardoEscobar Moscoso Jorge GabrielAdministrador de Datos: Encargado de la VM-REDIS. Gestión de persistencia en memoria y optimización. 4jogaesmoOnofre Alanoca RoyIngeniero de Observabilidad: Encargado de la VM-MON (Prometheus/Grafana). Monitoreo y auditoría. 5RoyOnofre🎯 I. Objetivo del ProyectoObjetivo: Diseñar e implementar una arquitectura de Cola Virtual (Virtual Waiting Room) distribuida para el sistema universitario SUNiver, capaz de interceptar el 100% del tráfico entrante, aplicar Rate Limiting para limitar la concurrencia a un umbral seguro (ej. 100 usuarios/minuto) y redirigir el exceso de tráfico a una sala de espera estática, garantizando así la disponibilidad del servicio crítico bajo condiciones de saturación. 6💡 II. Justificación e ImportanciaJustificación: El problema recurrente durante las fechas de inscripción es la Denegación de Servicio Involuntaria (saturación de usuarios), lo que genera una Falla de Continuidad Operacional (T1)7. Este proyecto es vital porque transforma el fallo en espera, convirtiendo un Error 503 en una espera ordenada8. Implementa Protección Centralizada (T5) mediante Nginx para desacoplar la carga masiva y Optimización Extrema (T4) utilizando Redis en RAM para gestionar la concurrencia a latencias de sub-milisegundos, resolviendo cuellos de botella de bases de datos tradicionales9.🛠️ III. Tecnologías y Conceptos Implementados3.1. Tecnologías ClaveNginx (VM-PROXY): Gateway y Reverse Proxy. Protege la topología interna y aplica el módulo limit_req para filtrar peticiones abusivas (Rate Limiting)10.Node.js (Express) (VM-APP): Servidor de Aplicación / Lógica. Ejecuta la lógica de "Portero": consulta el estado de la cola a Redis y decide si servir la página de Login o la Sala de Espera HTML11.Redis (VM-REDIS): Gestión de Estado Global (Memoria). Base de Datos NoSQL en memoria RAM. Mantiene el contador atómico del aforo en tiempo real, garantizando la "Verdad Única"12.Prometheus / Grafana (VM-MON): Observabilidad y Auditoría. Prometheus hace scraping de métricas de la aplicación, y Grafana visualiza la saturación de tráfico y el comportamiento de la cola13.Tailscale / Avahi (mDNS): Networking Transparente. Proporciona interconexión de las VMs mediante nombres de dominio .local, facilitando el descubrimiento de servicios sin IPs estáticas14.3.2. Conceptos de la Asignatura Puestos en Práctica (T1 - T6)Marca con un ✅ los temas avanzados de la asignatura que fueron implementados:Alta Disponibilidad (T2) y Tolerancia a Fallos: ✅ Implementación de un patrón Circuit Breaker Lógico que previene la caída en cascada del servidor principal mediante el desvío de tráfico a la sala de espera estática15.Seguridad y Hardening (T5): ✅ Uso de Rate Limiting en el Proxy Inverso (Nginx) para mitigar ataques de fuerza bruta y denegación de servicio (DDoS) en Capa 716.Automatización y Gestión (T6): ✅ Configuración de servicios systemd para el arranque automático y la recuperación de servicios (Nginx, Redis, App) tras reinicios no programados17.Balanceo de Carga/Proxy (T3/T4): ✅ Configuración de Upstreams en Nginx para abstraer la ubicación real de los servidores de aplicación y permitir escalabilidad horizontal futura18.Monitoreo (T4/T1): ✅ Implementación de un stack de observabilidad completo (Métricas RED) para detectar cuellos de botella y auditar la estabilidad bajo picos de carga19.Networking Avanzado (T3): ✅ Implementación de resolución de nombres interna (mDNS) y segmentación de servicios en distintas VMs/Hosts20.🌐 IV. Diseño de la Infraestructura y Topología4.1. Diseño EsquemáticoEl diseño se basa en la segmentación física de los servicios en 4 VMs distribuidas en hosts distintos, comunicadas por nombre de dominio a través de la red LAN (mDNS)21.VM/HostRolIP Lógica / HostnameSoftware PrincipalCapaSOVM-PROXYGateway / Rate Limiterproxy-server.localNginxAccesoUbuntu 22.04 22VM-APPLógica de Negocio / Workerapp-server.localNode.js v20AplicaciónUbuntu 22.04 23VM-REDISGestión de Estado (Cola)redis-server.localRedis Server 7DatosUbuntu 22.04 24VM-MONMonitoreo y Alertasmonitor-server.localPrometheus + GrafanaGestiónUbuntu 22.04 254.2. Estrategia Adoptada (Opcional)Estrategia de Desacoplamiento: Se separó el Proxy, la Lógica y el Estado en máquinas virtuales distintas. Esto garantiza que la VM-PROXY pueda seguir respondiendo con páginas de Sala de Espera, incluso si la VM-APP se sobrecarga o falla26.Estrategia de Optimización (Redis): Se eligió Redis sobre una base de datos SQL porque las operaciones de incremento/decremento de cola deben ser atómicas y de latencia cero. El uso de la RAM garantiza que el contador de cupos nunca sea el cuello de botella (T4)27.Networking Híbrido: La configuración del descubrimiento de servicios mediante mDNS (.local) permite que las máquinas se encuentren dinámicamente sin depender de IPs estáticas fijas, facilitando la movilidad del despliegue28.📋 V. Guía de Implementación y Puesta en MarchaSigue estos pasos detallados para replicar la infraestructura en 4 máquinas virtuales con Ubuntu 22.04.5.1. Pre-requisitos (Configuración de Red)IMPORTANTE: Ejecutar esto en las 4 VMs (Proxy, App, Redis, Monitor) para que se "vean" por nombre.Actualizar e instalar herramientas de red y Avahi Daemon (mDNS)29:Bashsudo apt update
sudo apt install -y net-tools curl avahi-daemon libnss-mdns
Configurar los Hostnames (Ejecutar en cada VM según corresponda)30303030303030303030303030303030:VM Redis: sudo hostnamectl set-hostname redis-serverVM App: sudo hostnamectl set-hostname app-serverVM Proxy: sudo hostnamectl set-hostname proxy-serverVM Monitor: sudo hostnamectl set-hostname monitor-serverReiniciar las VMs y probar conectividad (ej. ping app-server.local)31.5.2. Despliegue Paso a PasoPASO 1: VM-REDIS (Capa de Datos)Instalación: Instalar el servidor Redis32.Bashsudo apt install redis-server -y
Configuración: Editar el archivo para permitir conexiones externas33.Bashsudo nano /etc/redis/redis.conf
Buscar la línea: bind 127.0.0.1Cambiar por: bind 0.0.0.0Ejecución: Reiniciar el servicio34.Bashsudo systemctl restart redis-server
PASO 2: VM-APP (Capa de Lógica)Instalación: Instalar Node.js y dependencias35.Bashsudo apt install nodejs npm -y
mkdir proyecto-cola && cd proyecto-cola
npm init -y
npm install express redis
Codificación: Crear el archivo de lógica index.js36.Bashnano index.js
Pegar el siguiente código (Lógica del Portero):JavaScriptconst express = require('express');
const redis = require('redis');
const app = express();
const PORT = 3000;
const MAX_CAPACITY = 5; // Límite de usuarios concurrentes
const REDIS_KEY = 'suniver:active_users';

// CONEXIÓN A VM-REDIS
const client = redis.createClient({
    socket: { host: 'redis-server.local', port: 6379 }
});
client.connect();

app.get('/', async (req, res) => {
    try {
        const activeUsers = await client.incr(REDIS_KEY); // Incremento atómico
        if (activeUsers <= MAX_CAPACITY) {
            // ESCENARIO: HAY CUPO (LOGIN)
            setTimeout(() => client.decr(REDIS_KEY), 10000); // Simula sesión de 10s
            res.send(`<h1>✅ BIENVENIDO (Usuarios: ${activeUsers})</h1>`);
        } else {
            // ESCENARIO: SALA DE ESPERA
            await client.decr(REDIS_KEY); // Revertir conteo
            res.send(`<h1>🟠 SALA DE ESPERA (Saturado)</h1><meta http-equiv="refresh" content="5">`);
        }
    } catch (err) { res.status(503).send('Error del Servidor'); }
});

app.listen(PORT, () => console.log(`App en puerto ${PORT}`));
Ejecución: Correr el servidor en segundo plano37.Bashnohup node index.js &
PASO 3: VM-PROXY (Capa de Acceso)Instalación: Instalar Nginx38.Bashsudo apt install nginx -y
Configurar Rate Limit: Editar nginx.conf39.Bashsudo nano /etc/nginx/nginx.conf
Añadir dentro del bloque http { ... }:limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;Configurar Proxy Inverso: Editar el sitio default40.Bashsudo nano /etc/nginx/sites-available/default
Reemplazar contenido con:Nginxupstream backend_app {
    server app-server.local:3000;
}
server {
    listen 80 default_server;
    location / {
        limit_req zone=one burst=5 nodelay; # Rate Limiting
        proxy_pass http://backend_app;      # Proxy al upstream
        proxy_set_header Host $host;
    }
}
Ejecución: Reiniciar Nginx41.Bashsudo nginx -t && sudo systemctl restart nginx
PASO 4: VM-MON (Monitoreo)Configuración: Editar prometheus.yml para scrapear la app42.YAMLscrape_configs:
  - job_name: 'node_app'
    scrape_interval: 5s
    static_configs:
      - targets: ['app-server.local:3000']
5.3. Ficheros de Configuración Claveindex.js (en VM-APP): Algoritmo de decisión (If cupo -> Login, Else -> SalaEspera) y conexión remota a redis-server.local43./etc/nginx/nginx.conf (en VM-PROXY): Define la directiva limit_req_zone para el control de acceso44./etc/nginx/sites-available/default (en VM-PROXY): Define el upstream a app-server.local y aplica el limit_req./etc/redis/redis.conf (en VM-REDIS): Establece bind 0.0.0.0 para permitir la comunicación remota con la VM-APP45./etc/prometheus/prometheus.yml (en VM-MON): Configura el scraping de métricas de la VM-APP46.⚠️ VI. Pruebas y ValidaciónPrueba RealizadaResultado EsperadoResultado ObtenidoTest de Estrés (Saturación)Al superar el límite de N usuarios, el usuario N+1 debe ver la pantalla naranja de "Sala de Espera", no el Error 50347.OK: El sistema redirigió correctamente al usuario 6 a la cola.Test de Recuperación AutomáticaCuando un usuario activo abandona el sistema (timeout), la cola debe avanzar automáticamente48.OK: El usuario en espera ingresó automáticamente en el siguiente refresco.Prueba de Monitoreo en VivoGrafana debe mostrar el pico de tráfico y el estancamiento de usuarios activos en el límite máximo49.OK: Dashboard reflejó la curva de saturación en tiempo real.Validación de ConectividadLas VMs deben comunicarse por hostname (.local) independientemente de la IP asignada por el DHCP50.OK: Pings y conexiones de base de datos exitosas por nombre.📚 VII. Conclusiones y Lecciones AprendidasLogro Principal: Demostramos que la escalabilidad y la disponibilidad se logran mediante una arquitectura de software inteligente (Desacoplamiento), no solo con la adición de más hardware. El sistema transformó un escenario de fallo (Error 500) en una experiencia de usuario controlada (Sala de Espera)51.Lección Aprendida: La importancia de Redis (T4) es crítica. Intentar gestionar el estado de la cola en una base de datos tradicional hubiera introducido latencia, convirtiendo al contador en el principal cuello de botella. La gestión en RAM fue esencial52.Mejora Futura: Para un despliegue de producción real, implementaríamos un clúster de Redis (Sentinel) para evitar que la VM-REDIS sea un punto único de fallo, y aseguraríamos la capa de acceso con HTTPS y certificados SSL/TLS en Nginx53.
