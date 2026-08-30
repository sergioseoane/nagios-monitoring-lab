# nagios-monitoring-lab

Laboratorio de monitorización con Nagios Core, vigilando un servidor
Linux y la base de datos del proyecto [`postgres-retail-admin`](../postgres-retail-admin)
— conectando ambos proyectos entre sí, en vez de dejarlos sueltos.

## Qué vigila

| Servicio | Qué comprueba | Umbrales |
|---|---|---|
| Espacio en disco raíz | `/` | Warning 20%, Critical 10% libre |
| Espacio en disco de backups | `/home` | Warning 30%, Critical 15% libre (más exigente, para detectar antes de que fallen los backups) |
| Carga de CPU | Media de carga (1/5/15 min) | Pensado para una instancia pequeña (1-2 vCPU) |
| PostgreSQL - retail_demo | Conexión real a la base de datos del otro proyecto | Usa el rol `retail_readonly` (menor privilegio, no el superusuario) |


## Cómo probarlo

### A) Uso real: dentro de Docker Compose (portfolio-infra)

Este proyecto se monta como parte de
[`portfolio-infra`](../portfolio-infra), que ya incluye el servicio
`nagios` apuntando a `config/` de aquí. Desde `portfolio-infra`:

```bash
docker compose up -d
docker compose exec nagios /opt/nagios/libexec/check_pgsql \
  -H postgres -d retail_demo -l retail_readonly -p TU_CONTRASENA
```

### B) Prueba rápida y aislada (instalación nativa, para validar solo sintaxis)

```bash
sudo apt install nagios4 monitoring-plugins-basic monitoring-plugins-contrib
sudo cp config/*.cfg /etc/nagios4/conf.d/
sudo nagios4 -v /etc/nagios4/nagios.cfg
sudo systemctl restart nagios4
```

La interfaz web queda en `http://localhost/nagios4` (usuario/contraseña
configurados con `htpasswd` durante la instalación). Útil para probar
cambios de sintaxis rápido, sin depender de Docker.

## Estructura

```
config/
├── hosts.cfg      # el servidor a vigilar
├── services.cfg   # los 4 checks (cada uno con su contact_groups)
├── commands.cfg   # comando personalizado para el check de Postgres
└── contacts.cfg   # quién recibe las notificaciones
```


## Mejoras de nivel "producción" en `docker-compose.yml` (portfolio-infra)

- **Volumen `nagios-data`**: el historial de estados y logs de Nagios
  sobrevive a un `docker compose down` normal (sin `-v`), en vez de
  perderse cada vez que se recrea el contenedor.
- **Credenciales del panel web por variable de entorno**: nunca se
  dejan las de fábrica de la imagen (`nagiosadmin`/`nagios`).
- **`mem_limit`**: límite de memoria explícito, coherente con el
  presupuesto calculado para una instancia de 1GB.
- **`depends_on` con `condition: service_healthy`**: Nagios no arranca
  hasta que Postgres confirma (via `healthcheck`) que ya acepta
  conexiones, evitando una ventana de checks fallidos nada más
  arrancar todo junto.
