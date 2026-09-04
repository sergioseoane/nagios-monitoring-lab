# nagios-monitoring-lab

Laboratorio de monitorización de infraestructura con **Nagios Core**,
integrado con una base de datos PostgreSQL real y desplegado en
contenedores, replicando el diseño de un sistema de monitorización
de producción a pequeña escala.

## Objetivo del laboratorio

Diseñar e implementar un sistema de monitorización funcional que
cubra tres niveles habituales en un entorno real:

1. **Infraestructura** — estado del propio servidor (disco, carga).
2. **Aplicación** — disponibilidad de un servicio dependiente
(conexión a la base de datos de negocio).
3. **Operación** — gestión activa de incidencias (reconocimiento,
ventanas de mantenimiento), no solo detección pasiva de fallos.

El objetivo no era levantar Nagios "por levantarlo", sino integrarlo
con una infraestructura real ya existente ([`postgres-retail-admin`](../postgres-retail-admin))
y con el propio flujo de despliegue del portfolio
([`portfolio-infra`](../portfolio-infra)), tal como se integraría una
herramienta de monitorización en un entorno de trabajo real.

## Arquitectura

```
┌─────────────────────────┐        ┌──────────────────────────┐
│   servidor-portfolio     │        │   postgres (contenedor)   │
│   (host monitorizado)     │        │                           │
│                          │        │  retail_readonly (rol de  │
│  - Espacio en disco       │───────▶│  solo lectura, sin        │
│  - Carga de CPU           │  check │  privilegios de escritura)│
│                          │        │                           │
└─────────────────────────┘        └──────────────────────────┘
            ▲
            │ vigilado por
            │
   ┌─────────────────┐
   │   Nagios Core     │
   │   (contenedor)     │
   │                    │
   │ hosts / services /  │
   │ commands / contacts  │
   └─────────────────┘
```

Tanto Nagios como PostgreSQL viven en una red interna de Docker
(`backend-net`) sin ningún puerto publicado hacia internet — la única
forma de acceder al panel es mediante AWS Systems Manager Session
Manager (sin necesidad de abrir ningún puerto de entrada, ni
siquiera SSH), nunca de forma
pública, siguiendo el mismo criterio de seguridad aplicado al resto
del portfolio.

## Competencias técnicas aplicadas

* **Administración de sistemas y contenedores**: orquestación con
Docker Compose y segmentación de red interna (`edge-net` /
`backend-net`) para aislar los servicios de gestión del tráfico
público.
* **Configuración de Nagios Core**: hosts, servicios, comandos,
plantillas y contactos, con umbrales de alerta ajustados por tipo
de recurso.
* **Monitorización de bases de datos con mínimo privilegio**:
comprobación de disponibilidad de PostgreSQL mediante un rol de
solo lectura dedicado, no el superusuario.
* **Gestión de secretos**: credenciales fuera del código, mediante
variables de entorno y archivos de macros no versionados.
* **Automatización de operaciones (ITOps/NOC)**: scripts que
reconocen incidencias y programan ventanas de mantenimiento a
través del motor de comandos externos de Nagios.

## Estructura del repositorio

```
config/             → hosts, servicios, comandos y contactos de Nagios
config-overrides/   → anulación de la configuración de ejemplo de la imagen base
scripts/            → automatización operativa (Acknowledge, Scheduled Downtime)
resource.cfg.example → plantilla de credenciales (sin datos reales)
```

## Cómo ejecutarlo

Este proyecto no se ejecuta de forma aislada — se integra dentro de
[`portfolio-infra`](../portfolio-infra), que orquesta Nagios junto al
resto de la infraestructura:

```bash
cp resource.cfg.example resource.cfg
# Rellenar $USER10$ con la misma contraseña que RETAIL_READONLY_PASSWORD
# del .env de portfolio-infra

cd ../portfolio-infra
docker compose up -d nagios
```

Validación de la configuración, previa a cualquier despliegue:

```bash
docker compose exec nagios /opt/nagios/bin/nagios -v /opt/nagios/etc/nagios.cfg
```

## Automatización de incidencias, en la práctica

```bash
# Reconocer una incidencia activa (evita notificaciones repetidas
# mientras ya se está investigando):
docker compose exec nagios /opt/nagios/scripts-custom/acknowledge_problem.sh \
  servidor-portfolio "Carga de CPU" "En investigación"

# Programar una ventana de mantenimiento antes de una intervención
# planificada (silencia alertas durante el tiempo indicado):
docker compose exec nagios /opt/nagios/scripts-custom/schedule_downtime.sh \
  servidor-portfolio "PostgreSQL - retail_demo" 10 "Mantenimiento planificado"
```


## Proyectos relacionados

* [`postgres-retail-admin`](../postgres-retail-admin) — la base de
datos monitorizada por este laboratorio.
* [`portfolio-infra`](../portfolio-infra) — la infraestructura
completa donde se despliega, incluida la segmentación de red.

