### Root — Escalado de Privilegios en Linux

![License](https://img.shields.io/badge/plataforma-Linux-blue.svg)
![C](https://img.shields.io/badge/C-70.7%25-blue.svg)
![Shell](https://img.shields.io/badge/shell-24.7%25-green.svg)
![Python](https://img.shields.io/badge/python-4.6%25-yellow.svg)
![Status](https://img.shields.io/badge/status-activo-brightgreen.svg)

<p align="center">
  <img src="banner.png" alt="hackingyseguridad.com" />
</p>

Recopilación de **técnicas y exploits públicos (PoC)** para escalado de privilegios a **root** en sistemas Linux: desde comprobaciones manuales clásicas (SUID, sudo) hasta pruebas de concepto de CVEs concretas del kernel y de utilidades del sistema como `sudo` o `pkexec`.

---

### Tabla de contenidos

- [¿Qué es el escalado de privilegios?](#-qué-es-el-escalado-de-privilegios)
- [¿Qué incluye este repositorio?](#-qué-incluye-este-repositorio)
- [Instalación](#-instalación)
- [Técnica manual: binarios SUID](#-técnica-manual-binarios-suid)
- [Catálogo de exploits (PoC) por CVE](#-catálogo-de-exploits-poc-por-cve)
- [Otras carpetas del repositorio](#-otras-carpetas-del-repositorio)
- [Otras CVE relevantes de `sudo` (referencia)](#-otras-cve-relevantes-de-sudo-referencia)
- [Buenas prácticas de mitigación](#️-buenas-prácticas-de-mitigación)
- [Aviso legal / Uso responsable](#️-aviso-legal--uso-responsable)
- [Licencia](#-licencia)
- [Contribuciones](#-contribuciones)

---

### el escalado/elevacion de privilegios

El **escalado de privilegios** (*privilege escalation*) es la técnica mediante la cual un atacante que ya tiene acceso limitado a un sistema (usuario sin privilegios) consigue elevar su nivel de acceso hasta obtener permisos de **root/administrador**. En Linux, los vectores más comunes incluyen:

- Binarios con el bit **SUID** mal configurados.
- Vulnerabilidades en el **kernel** (corrupción de memoria, condiciones de carrera, *use-after-free*).
- Fallos en utilidades privilegiadas como **`sudo`** o **`pkexec`** (Polkit).
- Malas configuraciones de permisos, tareas cron, o servicios ejecutados como root.

---

### ¿Qué incluye este repositorio?

| Categoría | Contenido |
|---|---|
| PoC por CVE | Carpetas independientes con exploits públicos para CVEs concretas del kernel y de `sudo`/Polkit |
| Scripts de auditoría | `audit.sh`, `poc.sh`, `prueba.sh`, `kernel3.sh`, `root.sh` — comprobación y explotación |
| Referencia documental | Listado ampliado de CVEs históricas relacionadas con `sudo` |
| Recursos gráficos | Capturas ilustrativas de la explotación (`root.jpg`, `CVE-2025-32463.png`) |

---

### Instalación

```bash
cd /tmp
git clone https://github.com/hackingyseguridad/root
cd root
sh root.sh
```

---

###  Técnica manual: binarios SUID

Antes de recurrir a exploits específicos de CVE, la primera comprobación en cualquier auditoría de post-explotación es buscar binarios con el bit **SUID** activo (`Set User ID`, permiso `4000`), ya que se ejecutan con los privilegios de su propietario (a menudo root) independientemente de quién los invoque.

| Paso | Comando | Propósito |
|---|---|---|
| Buscar binarios SUID | `find / -perm -4000 2>/dev/null` | Localizar ejecutables que corren como root |
| Probar shell privilegiada | `/usr/bin/sudo /bin/sh -p` seguido de `whoami` | Verificar si se obtiene una shell con privilegios root |
| Persistencia (solo en pruebas autorizadas) | `sudo chmod +s /bin/sh` o `sudo chmod +s /bin/bash` | Activa el bit SUID sobre la shell: cualquier usuario que la ejecute obtiene privilegios de root |

<p align="center">
  <img src="root.jpg" alt="Elevación de privilegios a root vía sudo" width="500"/>
</p>

> El paso 3 (persistencia) **modifica permisos del sistema** y solo debe usarse en entornos de laboratorio o auditorías con autorización expresa — nunca en sistemas de producción sin consentimiento.

---

### Exploits (PoC) por CVE

| CVE | Componente afectado | Versiones afectadas | Severidad / Impacto | Vector de explotación (resumen) |
|---|---|---|---|---|
| [CVE-2008-0600](https://nvd.nist.gov/vuln/detail/CVE-2008-0600) | Linux Kernel — `vmsplice()` | 2.6.17 – 2.6.24.1 | Alta — root local | Validación incorrecta de puntero de espacio de usuario en `vmsplice_to_pipe`; argumentos manipulados llevan a escalado de privilegios |
| [CVE-2019-14287](https://nvd.nist.gov/vuln/detail/CVE-2019-14287) | Sudo | Anteriores a 1.8.28 | Alta | Fallo en verificación de IDs de usuario (`sudo -u #-1` / `#4294967295`) permite eludir restricciones de políticas y ejecutar como root |
| [CVE-2021-3156](https://nvd.nist.gov/vuln/detail/CVE-2021-3156) *(“Baron Samedit”)* | Sudo | Anteriores a 1.9.5p2 | Crítica | Desbordamiento de heap en `sudoedit -s` con argumento terminado en `\`, permite escalar de usuario sin privilegios a root |
| [CVE-2021-4034](https://nvd.nist.gov/vuln/detail/CVE-2021-4034) *(“PwnKit”)* | Polkit `pkexec` | Mayoría de distribuciones Linux | Crítica | Validación incorrecta del número de parámetros; manipulación de variables de entorno fuerza ejecución de código arbitrario como root |
| [CVE-2022-0847](https://nvd.nist.gov/vuln/detail/CVE-2022-0847) *(“Dirty Pipe”)* | Linux Kernel (pipes) | ≥ 5.8 (antes de parches) | Crítica | Inicialización inadecuada de estructuras en pipes permite sobrescribir archivos de solo lectura (incluidos binarios SUID) |
| [CVE-2024-27397](https://nvd.nist.gov/vuln/detail/CVE-2024-27397) | Linux Kernel — `nf_tables` (netfilter) | Previas a parches | Alta | *Use-after-free* por elemento expirado en reglas `nftables`; condición de carrera que corrompe memoria del kernel |
| [CVE-2025-32463](https://nvd.nist.gov/vuln/detail/CVE-2025-32463) | Sudo — opción `--chroot` (`-R`) | 1.9.14 – 1.9.17 | Crítica (9.3) | `sudo` resuelve rutas vía `chroot()` mientras evalúa `sudoers`; un `/etc/nsswitch.conf` falso en el chroot fuerza la carga de una librería compartida maliciosa |
| [CVE-2026-24061](https://nvd.nist.gov/vuln/detail/CVE-2026-24061) | GNU InetUtils `telnetd` | 1.9.3 – 2.7 | Crítica — remoto | Validación incorrecta de la variable de entorno `USER`; `USER=-f root` fuerza a `/usr/bin/login` a omitir la autenticación y otorgar acceso root remoto |
| CVE-2026-31431 | *(carpeta incluida en el repositorio — pendiente de documentar en este README)* | — | — | Consulta el contenido de la carpeta [`CVE-2026-31431`](CVE-2026-31431) para el detalle del PoC |

<p align="center">
  <img src="CVE-2025-32463.png" alt="CVE-2025-32463 - sudo chroot privilege escalation" width="500"/>
</p>

>  A diferencia de las CVEs de Sudo/Polkit (que requieren solo ejecución local), **CVE-2026-24061** es explotable de forma **remota** vía Telnet — trátala con prioridad si el servicio está expuesto.

---

### Otras carpetas

| Carpeta / Script | Contenido |
|---|---|
| `Dirtt-Frag/` | PoC/herramientas relacionadas con vulnerabilidades de fragmentación en el manejo de memoria del kernel |
| `Kernel2.6/` | Exploits orientados a versiones del Kernel Linux 2.6.x (incluye contexto de CVE-2008-0600) |
| `cve_2026_24061/` | Material adicional / variante del PoC de CVE-2026-24061 |
| `audit.sh` | Script de auditoría para detectar vectores de escalado de privilegios en el sistema (SUID, sudo, kernel, etc.) |
| `kernel3.sh` | Comprobaciones/explotación orientadas a vulnerabilidades del kernel |
| `poc.sh` | Prueba de concepto genérica de escalado de privilegios |
| `prueba.sh` | Script de pruebas del repositorio |
| `root.sh` | Script principal de instalación/lanzamiento (ver [Instalación](#-instalación)) |

> Cada carpeta de CVE suele incluir su propio código fuente (C/Python/Shell) y, en muchos casos, instrucciones de compilación y uso específicas — revisa el `README` interno de cada carpeta si existe.

---

### Otras CVE relevantes de `sudo` (referencia)

Además de las PoC incluidas en el repositorio, esta es una lista de referencia de otras vulnerabilidades históricas de `sudo` relacionadas con escalado de privilegios, útil para contextualizar auditorías más amplias:

| CVE | Resumen |
|---|---|
| CVE-2025-32462 | Aplicación incorrecta de la opción `--host`, puede permitir bypass de restricciones de `sudoers` |
| CVE-2023-22809 | `sudoedit` malinterpreta argumentos extra de `EDITOR`/`SUDO_EDITOR`/`VISUAL`, facilitando escalación local |
| CVE-2023-28487 | `sudoreplay -l` no escapa correctamente caracteres de control, exponiendo posible abuso de sesión |
| CVE-2023-27320 | Doble liberación (*double free*) en el soporte de chroot por comando, puede permitir corrupción y escalada |
| CVE-2023-7090 | Manejo incorrecto de `ipa_hostname`, puede provocar retención inapropiada de privilegios |
| CVE-2021-23240 | `sudoedit` con SELinux puede escalar privilegios mediante enlace simbólico en archivos temporales |
| CVE-2021-23239 | Condición de carrera en `sudoedit` permite comprobar la existencia de directorios arbitrarios vía symlinks |
| CVE-2019-19234 | No bloquea correctamente usuarios bloqueados, permitiendo suplantación con privilegios `Runas ALL` |
| CVE-2019-19232 | Similar a CVE-2019-19234: abuso del permiso `Runas ALL` para suplantar a otros usuarios |
| CVE-2019-18634 | Desbordamiento de pila si `pwfeedback` está habilitado en `sudoers` |
| CVE-2017-1000368 | Manejo inseguro de archivos/rutas en versiones antiguas |
| CVE-2017-1000367 | Entrada de usuario mal gestionada permite bypass de controles y elevación de privilegios |
| CVE-2016-7091 | Configuración predeterminada en algunas distros permite escalada por mal control de reglas |
| CVE-2016-7076 | Bypass de la política `noexec` en versiones anteriores a 1.8.18p1 |
| CVE-2016-7032 | El módulo `sudo_noexec.so` permite eludir restricciones `noexec` |
| CVE-2015-8239 | Fallo en soporte SHA-2 del plugin `sudoers` que permite abuso de autenticación/privilegios |
| CVE-2015-5602 | `sudoedit` permite escalación local en versiones anteriores a 1.8.15 |
| CVE-2014-9680 | No valida adecuadamente la variable `TZ`, permitiendo escalada local |
| CVE-2014-0106 | `env_reset` deshabilitado puede derivar en fallos de seguridad |
| CVE-2013-2777 | Mala implementación de `tty_tickets` puede permitir explotación y escalada local |
| CVE-2013-2776 | Reglas `Runas` mal gestionadas en versiones antiguas permiten escalada |

---

#
[www.hackingyseguridad.com](http://www.hackingyseguridad.com/)
#
