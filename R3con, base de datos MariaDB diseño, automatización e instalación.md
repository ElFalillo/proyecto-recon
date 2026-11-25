# R3con, base de datos MariaDB: diseño, automatización e instalación

La base de datos de R3con pasa a MariaDB y se instala en la VM r3con-db (10.10.30.20). El diseño se ha revisado a partir de la web: login con usuario y contraseña, dashboard, calendario, historial de informes y avisos. La lógica repetitiva se ejecuta dentro de la propia base de datos con triggers, procedimientos y eventos.

## Instalación paso a paso en r3con-db

La instalación se hace en la VM r3con-db de Proxmox, con Ubuntu Server 24.04, que instala MariaDB 10.11. Los comandos se ejecutan por SSH desde r3con-client o desde la consola de la VM en la web de Proxmox. Los cuatro scripts SQL van en este orden: `01_schema.sql`, `02_logic.sql`, `03_users.sql` y `04_seed.sql`.

1. **Dar a la VM su IP fija.** Editad `/etc/netplan/50-cloud-init.yaml` (el nombre puede variar; mirad con `ls /etc/netplan`) y aplicad con `sudo netplan apply`:

   ```yaml
   network:
     version: 2
     ethernets:
       ens18:
         addresses: [10.10.30.20/24]
         routes:
           - to: default
             via: 10.10.30.1
         nameservers:
           addresses: [10.10.30.10]
           search: [r3con.test]
   ```

   Comprobad con `ip -br a` que la interfaz tiene 10.10.30.20 y con `ping -c 2 10.10.30.1` que llegáis al gateway.
2. **Instalar MariaDB.** Con la variable `UPDATES=1` del gateway activa, para que la VM salga a Internet:

   ```bash
   sudo apt update && sudo apt install -y mariadb-server
   mariadb --version    # debe indicar 10.11
   ```
3. **Asegurar la instalación.** Ejecutad `sudo mariadb-secure-installation` y responded: autenticación por socket de Unix para root, sí; cambiar la contraseña de root, no (root solo entra con `sudo`); borrar usuarios anónimos, sí; impedir el acceso remoto de root, sí; borrar la base de datos de pruebas, sí; recargar privilegios, sí.
4. **Configurar el servidor.** Editad `/etc/mysql/mariadb.conf.d/50-server.cnf` y, en la sección `[mysqld]`, dejad estas líneas:

   ```ini
   bind-address         = 10.10.30.20
   character-set-server = utf8mb4
   collation-server     = utf8mb4_unicode_ci
   event_scheduler      = ON
   ```

   `bind-address` hace que MariaDB escuche solo en la red de servidores y `event_scheduler` activa las tareas programadas de la base de datos. Reiniciad con `sudo systemctl restart mariadb` y comprobad con `sudo ss -ltnp | grep 3306` que escucha en 10.10.30.20.
5. **Activar el firewall local.** Es la segunda barrera, además de iptables en el gateway:

   ```bash
   sudo ufw default deny incoming
   sudo ufw allow from 10.10.40.0/24 to any port 22 proto tcp   # administración
   sudo ufw allow from 10.10.30.30 to any port 22 proto tcp     # r3con-backup
   sudo ufw allow from 10.10.20.10 to any port 3306 proto tcp   # solo r3con-web
   sudo ufw enable
   ```
6. **Copiar los scripts a la VM.** Desde el equipo donde estén, por ejemplo r3con-client: `scp 0*.sql usuario@10.10.30.20:~/r3con-db/`. Si están en GitHub, también vale `git clone` en la VM.
7. **Crear la base de datos y su lógica.** Dentro de `~/r3con-db`:

   ```bash
   sudo mariadb < 01_schema.sql
   sudo mariadb < 02_logic.sql
   ```
8. **Crear los usuarios de MariaDB.** Las contraseñas se pasan al ejecutar y no quedan escritas en ningún archivo. Empezad el comando con un espacio para que no se guarde en el historial de bash:

   ```bash
    sudo mariadb -e "SET @admin_pass='CONTRASEÑA_ADMIN'; SET @app_pass='CONTRASEÑA_API'; SET @backup_pass='CONTRASEÑA_COPIAS'; SOURCE 03_users.sql;"
   ```
9. **Cargar los datos de prueba.** `sudo mariadb < 04_seed.sql`. Los usuarios de la web se crean sin contraseña válida, para que no haya contraseñas en el repositorio.
10. **Dar contraseña a los usuarios de la web.** La contraseña se cifra con bcrypt fuera de la base de datos, igual que hará la API. En cualquier equipo con Python:

    ```bash
    pip install bcrypt
    python3 -c "import bcrypt, getpass; print(bcrypt.hashpw(getpass.getpass().encode(), bcrypt.gensalt(12)).decode())"
    ```

    Copiad el hash que devuelve y guardadlo en r3con-db: `sudo mariadb r3con -e "CALL sp_set_password('admin', 'HASH_COPIADO');"`. Repetid para `cliente.admin` y `cliente.tecnico`.
11. **Abrir el puerto en el gateway.** En `r3con-gw-iptables.sh`, la regla de la base de datos debe usar `--dport 3306`. Volved a ejecutar el script y guardad las reglas.
12. **Probar la conexión desde r3con-web.** Instalad el cliente con `sudo apt install -y mariadb-client` y ejecutad `mariadb -h 10.10.30.20 -u r3con_app -p r3con -e "SELECT * FROM v_dashboard_resumen;"`. Debe devolver una fila con 3 hosts, 9 servicios y 5 hallazgos abiertos.
13. **Programar las copias.** Desde r3con-backup, la copia debe incluir procedimientos, triggers y eventos, porque `mariadb-dump` no los incluye por defecto:

    ```bash
    ssh admin@10.10.30.20 "mariadb-dump -u r3con_backup -p'CONTRASEÑA_COPIAS' --single-transaction --routines --triggers --events r3con" > r3con-$(date +%F).sql
    ```

## Esquema entidad relación

El esquema muestra las 19 tablas con sus columnas, tipos y claves, y las 35 relaciones entre ellas. Se lee con la notación de pata de gallo: la pata de gallo marca el lado N y la doble barra el lado 1. Una línea discontinua indica una relación opcional, en la que la clave foránea admite NULL; por ejemplo, un aviso puede no estar ligado a ningún hallazgo. Las tablas con borde discontinuo son las intermedias de las relaciones N:M, y el color agrupa las tablas por bloque, igual que en el apartado siguiente.

&#91;image: Esquema entidad relación de la base de datos r3con en MariaDB\]

## Tablas y por qué existe cada una

La base de datos tiene 19 tablas en cuatro bloques. Cada tabla responde a una pantalla de la web o a un paso del flujo de análisis; si una tabla no la necesitaba nada de eso, se quitó. Respecto al diseño anterior hay cinco cambios que vienen de la web:

- El login es con **username**, no con email, y `users` guarda los intentos fallidos para bloquear la cuenta tras 5 errores.
- Nueva tabla `user_sessions` para poder cerrar sesión y revocar accesos.
- Nueva tabla `calendar_events` para el calendario del dashboard.
- `notifications` sirve ahora para la campana del portal y para Telegram.
- `support_messages` admite respuestas encadenadas, para el icono de mensajes.

### Organizaciones, usuarios y sesiones

| Tabla | Qué hace en la web | Por qué existe |
| --- | --- | --- |
| `organizations` | No se ve, pero filtra todo lo que ve cada cliente | Es la raíz de la separación de datos. Cada cliente ve solo sus equipos, hallazgos e informes |
| `users` | Pantalla de login y menú de usuario del dashboard | Guarda username, email, nombre, rol y el hash bcrypt de la contraseña. `failed_attempts` y `locked_until` bloquean la cuenta 15 minutos tras 5 intentos fallidos, contra ataques de fuerza bruta |
| `user_sessions` | Mantiene la sesión abierta y permite "Cerrar sesión" | La API da un token corto (JWT) y otro largo para renovarlo. Guardar el hash del token largo permite revocarlo: al cerrar sesión, al desactivar un usuario o si se roba el token |

### Políticas, agentes y escaneos

| Tabla | Qué hace en la web | Por qué existe |
| --- | --- | --- |
| `scan_policies` | Configuración de escaneos (solo administrador R3con) y origen de los escaneos del calendario | Define qué se puede escanear: puertos, perfil, máximo de equipos y periodicidad. `run_time` es la hora a la que se programa. Los agentes nunca reciben comandos libres |
| `policy_networks` | Rangos de cada política | Una política puede tener varios rangos permitidos y excluidos. Se guardan como red y prefijo (10.10.10.0 y 24) para que la base de datos pueda comprobar si una IP está dentro |
| `agents` | Tarjeta "Agentes" del dashboard y menú Agentes | Guarda cada contenedor (sensor y analizador), su versión, su estado y la última conexión. El token se guarda como hash, así un agente se puede revocar |
| `agent_policies` | No se ve | Tabla intermedia N:M: un agente puede tener varias políticas y una política varios agentes |
| `scans` | Tarjeta "Último escaneo" del dashboard | Registra cada ejecución: qué agente, con qué política, cuándo y con qué resultado. Es la prueba de que cada análisis estaba autorizado |

### Activos, servicios, reglas y hallazgos

| Tabla | Qué hace en la web | Por qué existe |
| --- | --- | --- |
| `assets` | Menú Activos y contador de hosts del dashboard | Un registro por equipo descubierto. La criticidad y la exposición alimentan el cálculo de riesgo |
| `services` | Detalle de cada activo y contador de servicios | Un registro por puerto abierto, con producto y versión. Muchos hallazgos afectan a un servicio concreto |
| `rules` | Título y recomendación de cada hallazgo | Catálogo de reglas del analizador. La recomendación se escribe una sola vez y la heredan todos los hallazgos de esa regla |
| `findings` | Menú Hallazgos, tabla "Hallazgos recientes" y gráfico por severidad | El dato central de la plataforma: qué regla saltó, en qué equipo y puerto, con qué evidencia, severidad, confianza, estado y puntuación de riesgo. Una clave única evita duplicados entre escaneos |
| `finding_events` | Historial dentro del detalle de un hallazgo | Guarda cada cambio de estado, quién lo hizo y por qué. Lo rellena un trigger, así no depende de que la API se acuerde |

### Informes, calendario, avisos, soporte y auditoría

| Tabla | Qué hace en la web | Por qué existe |
| --- | --- | --- |
| `reports` | Historial de informes y descarga de PDF | Un registro por PDF con su periodo, ruta y hash SHA-256, que demuestra que el archivo no se ha modificado. `requested_by` guarda quién pidió los informes bajo demanda |
| `report_findings` | Contenido de cada informe | Tabla intermedia N:M: un informe incluye muchos hallazgos y un hallazgo aparece en varios informes |
| `calendar_events` | Calendario del dashboard | Muestra escaneos e informes programados, mantenimientos y recordatorios. Los escaneos los crea un evento programado a partir de las políticas, y su estado se actualiza solo cuando el escaneo termina |
| `notifications` | Campana del portal y alertas de Telegram | Una sola tabla para los dos canales. Los triggers crean los avisos; la web marca como leídos los del portal y n8n envía los de Telegram pendientes |
| `support_messages` | Icono de mensajes y chat de soporte | Cada consulta y sus respuestas, enlazadas con `parent_id` |
| `audit_logs` | Solo para el administrador R3con | Registra logins, cambios y descargas. Un trigger impide modificar o borrar registros, para que sirva como evidencia |

## Automatización dentro de la base de datos

La lógica que se repite en cada escaneo, cada login o cada cambio de estado está en la base de datos y no en la API. Así se cumple siempre, aunque la API tenga un fallo o se escriba otra aplicación más adelante. En total hay 5 funciones, 16 procedimientos, 16 triggers, 7 vistas y 4 eventos programados, todos en `02_logic.sql`.

### Triggers: reacciones automáticas a cambios

Un trigger se ejecuta solo cuando se inserta, modifica o borra una fila de una tabla.

| Trigger | Cuándo salta | Qué hace |
| --- | --- | --- |
| `trg_users_bi`, `trg_users_bu` | Al crear o editar un usuario | Pasa username y email a minúsculas, para que `Admin` y `admin` sean el mismo usuario |
| `trg_users_au` | Al desactivar un usuario | Cierra todas sus sesiones abiertas |
| `trg_networks_bi`, `trg_networks_bu` | Al guardar un rango en una política | Rechaza redes públicas y normaliza la dirección (10.10.10.7/24 se guarda como 10.10.10.0/24) |
| `trg_agents_bu` | Cuando un agente envía señal de vida | Lo pasa de desconectado a conectado |
| `trg_findings_bi` | Al crear un hallazgo | Toma la severidad de la regla y calcula la puntuación de riesgo |
| `trg_findings_bu` | Al modificar un hallazgo | Exige justificación para aceptar un riesgo y recalcula el riesgo si cambia la severidad o la confianza |
| `trg_findings_ai` | Al crear un hallazgo crítico | Crea un aviso en la campana del portal y otro para Telegram |
| `trg_findings_au` | Al cambiar el estado de un hallazgo | Guarda el cambio en `finding_events` con el usuario y el comentario |
| `trg_scans_bu`, `trg_scans_au` | Al terminar un escaneo | Pone la fecha de fin, marca el escaneo como completado o fallido en el calendario y avisa si ha fallado |
| `trg_reports_ai` | Al registrar un informe | Avisa en el portal de que hay un informe nuevo y lo añade al calendario |
| `trg_audit_bu`, `trg_audit_bd`, `trg_events_bu` | Al intentar modificar o borrar evidencias | Lo impiden: auditoría e histórico son de solo inserción |

### Procedimientos: las acciones de la web y del agente

La API no escribe directamente en las tablas importantes: llama a un procedimiento que valida, escribe y audita en un solo paso.

| Procedimiento | Quién lo usa | Qué hace |
| --- | --- | --- |
| `sp_login_datos` | API, pantalla de login | Devuelve el hash de la contraseña y si el usuario está activo o bloqueado |
| `sp_login_ok` | API, tras comprobar la contraseña | Reinicia intentos, abre la sesión y la audita |
| `sp_login_fallido` | API, contraseña incorrecta | Suma un intento; al quinto bloquea la cuenta 15 minutos. Lo audita |
| `sp_sesion_valida`, `sp_logout` | API | Comprueba o cierra una sesión |
| `sp_set_password` | Administrador | Guarda un hash bcrypt y rechaza cualquier otro valor |
| `sp_agent_heartbeat` | Agente, cada minuto | Actualiza la última conexión y rechaza agentes revocados |
| `sp_iniciar_escaneo` | Agente | Abre un escaneo solo si el agente tiene esa política asignada y activa |
| `sp_registrar_activo` | Agente | Crea o actualiza un equipo. Rechaza cualquier IP fuera del alcance de la política |
| `sp_registrar_servicio` | Agente | Crea o actualiza un puerto de un equipo |
| `sp_registrar_hallazgo` | Analizador | Crea el hallazgo o actualiza el existente; si estaba resuelto y reaparece, lo reabre |
| `sp_finalizar_escaneo` | Agente | Cierra el escaneo y cuenta los equipos encontrados |
| `sp_cambiar_estado_hallazgo` | Web, detalle de hallazgo | Comprueba que el usuario es de la misma organización y cambia el estado |
| `sp_registrar_informe` | Worker de PDF | Registra el informe y enlaza los hallazgos de su periodo |
| `sp_descargar_informe` | Web, historial de informes | Comprueba el permiso, audita la descarga y devuelve la ruta del PDF |
| `sp_programar_escaneos` | Evento diario | Crea en el calendario los escaneos de los próximos días |

### Funciones

| Función | Qué calcula |
| --- | --- |
| `fn_risk_score` | La fórmula de riesgo: severidad × exposición × criticidad × confianza, de 1 a 108 |
| `fn_mask`, `fn_ip_in_network` | Si una IP está dentro de una red con su prefijo |
| `fn_is_private` | Si una red es privada (10/8, 172.16/12 o 192.168/16) |
| `fn_ip_autorizada` | Si una política permite escanear una IP: dentro de un rango permitido y fuera de los excluidos |

### Vistas: consultas listas para cada pantalla

| Vista | Pantalla de la web |
| --- | --- |
| `v_dashboard_resumen` | Tarjetas de hosts, servicios, hallazgos por severidad y último escaneo |
| `v_hallazgos` | Tabla de hallazgos con equipo, puerto, regla y recomendación |
| `v_estado_agentes` | Tarjeta de agentes |
| `v_historial_informes` | Historial de informes |
| `v_calendario` | Calendario |
| `v_avisos_portal` | Campana de avisos |
| `v_cola_telegram` | Cola que lee n8n para enviar las alertas |

### Eventos programados

El Event Scheduler de MariaDB ejecuta tareas a horas fijas sin cron ni n8n. Necesita `event_scheduler = ON`, que se activó en el paso 4 de la instalación.

| Evento | Cuándo | Qué hace |
| --- | --- | --- |
| `ev_agentes_desconectados` | Cada 5 minutos | Marca como desconectados los agentes sin señal en 10 minutos y avisa por portal y Telegram |
| `ev_programar_calendario` | Cada día a las 00:05 | Rellena el calendario con los escaneos de los próximos 14 días |
| `ev_recordatorio_pendientes` | Cada día a las 09:00 | Avisa de hallazgos críticos o altos sin revisar desde hace más de 3 días |
| `ev_limpiar_sesiones` | Cada día a las 03:00 | Borra sesiones caducadas o revocadas hace más de 7 días |

La base de datos registra y avisa, pero no ejecuta escaneos ni genera PDF. Lanzar Nmap lo hacen los agentes, generar el PDF lo hace el worker y enviar el Telegram lo hace n8n leyendo `v_cola_telegram`.

## Cómo usa la web la base de datos

La web nunca habla con MariaDB directamente. El navegador llama a la API FastAPI en r3con-web, y la API se conecta a r3con-db con el usuario `r3con_app`. Cada consulta de datos de cliente se filtra por el `organization_id` del usuario que ha iniciado sesión.

### Login

1. El usuario escribe username y contraseña en la pantalla de login.
2. La API llama a `sp_login_datos(username)`. Si no devuelve fila, el usuario no existe; si `puede_entrar` es 0 o `bloqueado` es 1, se rechaza el acceso.
3. La API compara la contraseña con `password_hash` usando bcrypt. La contraseña nunca llega a MariaDB.
4. Si es correcta, la API genera los tokens y llama a `sp_login_ok` con el hash SHA-256 del token de renovación. Si no lo es, llama a `sp_login_fallido`.
5. En las peticiones siguientes, la API valida el token corto por sí misma y, al renovarlo, llama a `sp_sesion_valida`. "Cerrar sesión" llama a `sp_logout`.

La respuesta al usuario es la misma si el username no existe o si la contraseña está mal ("Usuario o contraseña incorrectos"), para no revelar qué usuarios existen.

### Dashboard

| Elemento del wireframe | Consulta |
| --- | --- |
| Tarjeta "Último escaneo" y contadores | `SELECT * FROM v_dashboard_resumen WHERE organization_id = ?` |
| Hallazgos por severidad | Mismas columnas `criticos`, `altos`, `medios` y `bajos` de esa vista |
| Tarjeta de agentes | `SELECT * FROM v_estado_agentes WHERE organization_id = ?` |
| Tabla "Hallazgos recientes" | `SELECT * FROM v_hallazgos WHERE organization_id = ? ORDER BY last_seen DESC LIMIT 10` |
| Filtros Hoy, 7 días, 30 días, Histórico | La misma consulta con `AND last_seen >= NOW() - INTERVAL 7 DAY`, según la pestaña |
| Cambiar estado de un hallazgo | `CALL sp_cambiar_estado_hallazgo(id, usuario, estado, comentario, ip)` |
| Campana de avisos | `SELECT * FROM v_avisos_portal WHERE organization_id = ? AND (user_id IS NULL OR user_id = ?) ORDER BY created_at DESC` |
| Marcar aviso como leído | `UPDATE notifications SET status = 'leido', read_at = NOW() WHERE id = ?` |
| Icono de mensajes | `INSERT` y `SELECT` sobre `support_messages` |

### Calendario

El calendario pide los eventos del mes visible: `SELECT * FROM v_calendario WHERE organization_id = ? AND starts_at BETWEEN ? AND ?`. Los escaneos programados aparecen solos gracias al evento diario, pasan a completados o fallidos cuando termina el escaneo, y los informes se añaden en cuanto se generan. Un administrador de cliente puede añadir mantenimientos o recordatorios con un `INSERT` en `calendar_events`.

### Historial de informes

La lista sale de `SELECT * FROM v_historial_informes WHERE organization_id = ? ORDER BY created_at DESC`. Al pulsar "Descargar PDF", la API llama a `sp_descargar_informe(id, usuario, ip)`, que comprueba el permiso, audita la descarga y devuelve la ruta del archivo, y la API sirve el PDF. Para un informe bajo demanda, el worker genera el PDF y llama a `sp_registrar_informe`.

## Pruebas de verificación

Todos los scripts se han probado en MariaDB 10.11.14, la versión de Ubuntu 24.04: carga desde cero, datos de prueba, permisos, copia y restauración. Repetid estas pruebas en r3con-db y guardad capturas como evidencia para la memoria. Se ejecutan con `sudo mariadb r3con -e "..."`, salvo las de permisos, que van desde r3con-web con `r3con_app`.

| # | Prueba | Comando | Resultado esperado |
| --- | --- | --- | --- |
| 1 | Objetos creados | `SHOW FULL TABLES;` y `SHOW EVENTS;` | 19 tablas, 7 vistas y 4 eventos |
| 2 | Datos del dashboard | `SELECT * FROM v_dashboard_resumen WHERE organization_id = 2;` | 3 hosts, 9 servicios, 5 hallazgos: 1 crítico, 1 alto, 2 medios y 1 bajo |
| 3 | Riesgo automático | `SELECT hallazgo, risk_score FROM v_hallazgos ORDER BY risk_score DESC;` | Telnet con 108, el máximo posible |
| 4 | Histórico automático | `SELECT * FROM finding_events;` | 2 cambios de estado, con usuario y comentario |
| 5 | Avisos automáticos | `SELECT channel, title FROM notifications;` | Aviso del crítico en portal y Telegram, y avisos de informes nuevos |
| 6 | Calendario | `SELECT type, status, starts_at FROM v_calendario;` | Escaneo de hoy completado y escaneos programados de los próximos días |
| 7 | Red pública rechazada | `INSERT INTO policy_networks (policy_id, network, prefix, type) VALUES (1, '8.8.8.0', 24, 'permitido');` | Error: solo se permiten redes privadas |
| 8 | IP fuera de alcance | `CALL sp_registrar_activo(1, '10.10.10.1', 'gw', @x);` | Error: el gateway está excluido de la política |
| 9 | Aceptar sin justificar | `CALL sp_cambiar_estado_hallazgo(1, 3, 'aceptado', '', '10.10.40.103');` | Error: hay que escribir una justificación |
| 10 | Bloqueo de login | 5 veces `CALL sp_login_fallido('admin', '10.10.40.9');` y luego `CALL sp_login_datos('admin');` | `bloqueado` = 1 durante 15 minutos |
| 11 | Auditoría inalterable | `DELETE FROM audit_logs WHERE id = 1;` | Error: el registro de auditoría no se puede borrar |
| 12 | Mínimo privilegio | Con `r3con_app`: `UPDATE findings SET status = 'resuelto' WHERE id = 1;` | Error: comando denegado. El cambio solo es posible con `sp_cambiar_estado_hallazgo` |
| 13 | Agente desconectado | `UPDATE agents SET last_seen_at = NOW() - INTERVAL 20 MINUTE WHERE id = 1;` y esperar 5 minutos | El agente pasa a desconectado y aparece el aviso |
| 14 | Copia y restauración | `mariadb-dump` del paso 13, cargado en una base de datos vacía | La vista del dashboard devuelve los mismos datos que la original |

La prueba 14 es la que pide la actividad para validar las copias: una copia solo cuenta como buena cuando se ha restaurado con éxito.
