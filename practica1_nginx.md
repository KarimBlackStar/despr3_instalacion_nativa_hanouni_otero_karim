# Práctica: Instalación y Configuración de Servidor Web Nginx

## 1. Configuración de Puertos en VirtualBox
Comprobamos la configuración de red inicial de la máquina virtual y las reglas de reenvío de puertos.

![img.png](img.png)

## 2. Conexión SSH
Establecemos la conexión remota a la máquina virtual (Ubuntu Server) a través de la terminal del anfitrión utilizando SSH.

![img_1.png](img_1.png)

## 3. Resolución de Problemas e Instalación de Nginx
Durante el proceso inicial de instalación mediante el gestor de paquetes, surgieron conflictos de descarga debido a que la configuración del sistema conservaba el proxy del instituto.

![img_2.png](img_2.png)
![img_3.png](img_3.png)

Procedimos a reconfigurar los ajustes de red para eliminar la dependencia de dicho proxy.

![img_4.png](img_4.png)

Una vez resueltos los bloqueos de red, la instalación del paquete `nginx` se completó con éxito.

![img_5.png](img_5.png)

## 4. Verificación del Servicio Nginx
Tras la instalación, realizamos las comprobaciones pertinentes mediante `systemctl` para confirmar que el servicio se encuentra habilitado y en estado `active (running)`.

![img_6.png](img_6.png)
![img_7.png](img_7.png)

## 5. Exploración de los Archivos de Configuración
Exploramos la estructura de directorios que ha generado el paquete, prestando especial atención a las carpetas `/etc/nginx/sites-available` y `/etc/nginx/sites-enabled`.

![img_8.png](img_8.png)
![img_9.png](img_9.png)

## 6. Configuración del Cortafuegos (UFW)
Para asegurar el servidor, habilitamos el cortafuegos `ufw`. Previamente, permitimos las conexiones SSH para no perder el acceso remoto y abrimos el tráfico web configurando la regla para `Nginx HTTP`.

![img_10.png](img_10.png)
![img_11.png](img_11.png)
![img_12.png](img_12.png)

## 7. Despliegue del Sitio Web Estático y Permisos
Descomprimimos los archivos del sitio web de prueba y los transferimos a la máquina virtual. A continuación, los ubicamos en su directorio definitivo (`/var/www/sitio`) y aplicamos las políticas de seguridad: asignamos `root` como propietario y establecemos permisos estrictos (755 para directorios y 644 para archivos).

![img_13.png](img_13.png)
![img_14.png](img_14.png)
![img_15.png](img_15.png)
![img_16.png](img_16.png)

## 8. Creación y Activación del Virtual Host
Generamos el archivo de configuración para nuestro sitio web, lo vinculamos con un enlace simbólico y desactivamos el sitio por defecto de Nginx. 

![img_17.png](img_17.png)
![img_18.png](img_18.png)

Durante las pruebas desde el navegador del anfitrión, la conexión era rechazada (`ERR_CONNECTION_REFUSED`). Tras diagnosticar el problema, se determinó que faltaba la regla correcta en el reenvío de puertos de VirtualBox. Se configuró el puerto anfitrión 8081 para que apuntara al puerto 80 del invitado. Tras aplicar el cambio, el sitio web cargó correctamente, registrando los códigos de estado `200 OK` en las Herramientas de Desarrollo.

![img_19.png](img_19.png)
![img_20.png](img_20.png)
![img_21.png](img_21.png)

## 9. Diagnóstico y Resolución de Errores (Troubleshooting)
Provocamos intencionadamente fallos de configuración para analizar su impacto y su registro en los logs:
1.  **Error de sintaxis:** Modificamos un archivo de configuración introduciendo un error. Se verificó con `nginx -t` y se consultó `journalctl` tras un intento fallido de reinicio del servicio.
2.  **Error de permisos:** Retiramos los permisos de lectura a la hoja de estilos (`chmod 600`), lo que provocó un error `403 Forbidden` al solicitar el recurso, dejando constancia en `/var/log/nginx/error.log`.

![img_22.png](img_22.png)
![img_23.png](img_23.png)

## 10. Comprobación de Arranque Automático
Reiniciamos la máquina virtual (`sudo reboot`) para certificar que el servicio Nginx arranca de forma automática y desatendida. Esto es posible gracias a que el servicio se encuentra habilitado (`enabled`) en el gestor `systemd`.

![img_24.png](img_24.png)

---

## Parte Final: Nginx Nativo vs. Contenedores (Docker)

| Aspecto | Instalación nativa | Contenedor |
| :--- | :--- | :--- |
| **Orden de instalación** | `apt install nginx` | Descarga de la imagen base en `compose.yaml` o Dockerfile (`image: nginx`). |
| **Versión instalada (y quién la decide)** | La disponible en los repositorios oficiales de Ubuntu (lo decide el gestor de paquetes del SO). | La etiqueta (*tag*) específica declarada en la imagen (lo decide el desarrollador). |
| **Arrancar, detener, recargar** | `systemctl start / stop / reload nginx` | `docker start / stop` o recargar enviando la señal al proceso (`docker exec -it <nombre> nginx -s reload`). |
| **Arranque automático** | `systemctl enable nginx` (crea el enlace simbólico en systemd). | Política de reinicio en el compose (`restart: always` o `unless-stopped`). |
| **Archivo de configuración del sitio** | `/etc/nginx/sites-available/` (enlazado a `sites-enabled/`). | Normalmente en `/etc/nginx/conf.d/` dentro del contenedor. |
| **Raíz de documentos (Document Root)** | Físicamente en `/var/www/` o `/var/www/html`. | Donde se mapee el volumen del anfitrión (ej. montado en `/usr/share/nginx/html`). |
| **Usuario de los procesos trabajadores** | `www-data` | `nginx` (por defecto en la imagen oficial). |
| **Ubicación de los logs** | Archivos físicos en `/var/log/nginx/` (access.log y error.log). | Salida estándar capturada y gestionada por el daemon de Docker (`docker logs`). |
| **Cómo se abre el puerto** | Reglas manuales en el cortafuegos del SO (`ufw allow 'Nginx HTTP'`). | Mapeo de puertos (`ports: - "80:80"`), que crea automáticamente las reglas iptables saltándose UFW. |
| **Cómo se actualiza a una versión nueva** | `apt update` y `apt upgrade nginx`. | Cambiando la etiqueta de la imagen en el archivo y recreando el contenedor (`docker compose up -d`). |
| **Cómo se deja todo como estaba** | Desinstalando (`apt purge nginx`), borrando dependencias huerfanas y eliminando directorios de configuración manualmente. | Destruyendo el contenedor, su red y los volúmenes anónimos asociados (`docker compose down -v`). |

### Respuestas razonadas:

1.  **¿Qué pasos has tenido que hacer aquí que el contenedor te daba resueltos?**
    En esta práctica nativa hemos tenido que instalar dependencias a mano, configurar el arranque automático en `systemd`, crear toda la estructura de carpetas, asignar minuciosamente los propietarios (`chown`) y permisos (`chmod`) por seguridad, y abrir explícitamente el puerto en el cortafuegos UFW. El contenedor te da todo este entorno base ya preconfigurado y aislado por defecto.

2.  **Un compañero necesita tener exactamente el mismo servidor. ¿Qué tendría que hacer en cada caso?**
    *   *En nativo:* Tendría que instalar la misma versión del sistema operativo base, ejecutar el mismo historial de comandos `apt`, configurar su UFW, y crear y copiar a mano todos los archivos de configuración y la web en las rutas exactas.
    *   *En contenedores:* Solo necesitaría que se le proporcione el archivo `compose.yaml` y la carpeta con la web. Con tener Docker instalado y lanzar `docker compose up -d`, tendría una réplica 100% exacta funcionando en segundos, sin importar su sistema operativo anfitrión.

3.  **¿En qué situación elegirías una instalación nativa y en cuál, contenedores?**
    *   *Nativa:* Se elegiría para exprimir al máximo el rendimiento directo del hardware del servidor (sin capas de abstracción por medio) o si la aplicación necesita integraciones muy profundas y específicas con el núcleo del sistema operativo.
    *   *Contenedores:* Se elegirían para entornos de desarrollo (garantizando consistencia entre máquinas), para desplegar y migrar servicios rápidamente, o para montar arquitecturas basadas en microservicios que requieran gran escalabilidad.