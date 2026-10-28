# Taller 3: Servidor web con Nginx en Oracle Cloud (OCI)

Publicación de un sitio web estático (HTML + CSS + JavaScript) en una máquina virtual de Oracle Cloud, servido con Nginx y accesible desde internet con un dominio de DuckDNS.

 **Sitio publicado:** http://servertaller3.duckdns.org

---

## Datos del despliegue

| Elemento | Valor |
|---|---|
| Proveedor | Oracle Cloud Infrastructure (Brazil East - São Paulo) |
| Compartimento | `Infraestructura_y_Cloud` |
| Nombre de la VM | `Servidor-Nginx-Taller3` |
| Sistema operativo | Oracle Linux 9 |
| Shape | `VM.Standard.E3.Flex` (1 OCPU, 16 GB RAM) |
| IP pública | `137.131.240.182` |
| IP privada | `10.0.0.105` |
| Servidor web | Nginx |
| Carpeta del sitio | `/var/www/misitio` |
| Dominio | `servertaller3.duckdns.org` |

---

## 1. Crear la instancia

En la consola de OCI: **Compute → Instances → Create instance**, con Oracle Linux 9, IP pública y llave SSH (`llave-web1.key`).

La instancia aparece en estado **Running** con su IP pública y privada:

![Instancia creada en OCI](imagenes/01-instancia-running.png)

---

## 2. Conexión por SSH desde Cloud Shell

### 2.1 Primer intento (con errores)

Al principio aparecieron varios problemas, que sirvieron para aprender:

- Se usó el nombre de llave equivocado (`llave-web.key`), pero el archivo subido se llamaba `llave-web1.key`.
- Se intentó entrar con el usuario `ubuntu`, y en Oracle Linux el usuario correcto es **`opc`**.
- Se ejecutó `sudo dnf install` en la propia Cloud Shell en lugar de en la VM, y falló.

![Errores del primer intento](imagenes/02-errores-ssh.png)

### 2.2 Conexión correcta

Se dieron permisos seguros a la llave y se entró con el usuario `opc`:

```bash
chmod 600 ~/llave-web1.key
ssh -i ~/llave-web1.key opc@137.131.240.182
```

---

## 3. Instalar Nginx y Git

Ya dentro de la VM (el prompt cambia a `[opc@servidor-nginx-taller3 ~]$`):

```bash
sudo dnf install -y nginx git
```

![Instalación de nginx y git](imagenes/03-instalacion-nginx-git.png)

---

## 4. Crear el sitio web

### 4.1 Estructura de archivos

```bash
sudo mkdir -p /var/www/misitio
sudo nano /var/www/misitio/index.html
sudo nano /var/www/misitio/style.css
sudo nano /var/www/misitio/script.js
```

El sitio tiene tres archivos:

| Archivo | Función |
|---|---|
| `index.html` | Estructura: encabezado, menú (Inicio, Servicios, Contacto), 3 secciones y pie de página |
| `style.css` | Estilos y diseño adaptativo |
| `script.js` | Interactividad con JavaScript |

### 4.2 `index.html`

![Código de index.html](imagenes/04-index-html.png)

### 4.3 `script.js`

Muestra un mensaje en verde con la fecha actual al pulsar el botón:

```javascript
document.getElementById('miBoton').addEventListener('click', function() {
    const mensaje = document.getElementById('mensajeJS');
    mensaje.textContent = "¡Funcionalidad JS ejecutada con éxito! Fecha actual: " + new Date().toLocaleDateString();
    mensaje.style.color = "green";
    mensaje.style.fontWeight = "bold";
});
```

> **Error común:** al pegar el código JavaScript directamente en la terminal (en vez de dentro de `nano`), Bash lo interpreta como comandos y aparecen errores como `syntax error near unexpected token`. Siempre hay que abrir primero el editor con `sudo nano ...` y pegar el contenido ahí.

### 4.4 Permisos

```bash
sudo chown -R nginx:nginx /var/www/misitio
sudo chmod -R 755 /var/www/misitio
```

---

## 5. Configurar Nginx

Se creó un archivo de configuración propio para el sitio:

```bash
sudo nano /etc/nginx/conf.d/misitio.conf
```

Contenido base:

```nginx
server {
    listen 80;
    server_name servertaller3.duckdns.org;

    root /var/www/misitio;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Comprobar la sintaxis y reiniciar el servicio:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

La salida debe indicar `syntax is ok` y `test is successful`:

![Prueba de configuración y reinicio de Nginx](imagenes/05-nginx-test-restart.png)

---

## 6. Firewall y SELinux

Abrir el puerto 80 (HTTP) en el firewall de la VM:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

Verificar los archivos del sitio:

```bash
ls -l /var/www/misitio/
```

Si Nginx devuelve error 403 al leer los archivos, SELinux puede estar bloqueando el acceso. Para probar, se puso en modo permisivo:

```bash
sudo setenforce 0
sudo systemctl restart nginx
```

![Historial de comandos: firewall, permisos y SELinux](imagenes/06-firewall-selinux.png)

> **Nota:** además del firewall de la VM, la Security List de la subred en OCI debe tener una regla de entrada (Ingress) para el puerto 80 (TCP, origen `0.0.0.0/0`).

---

## 7. Dominio con DuckDNS

1. Iniciar sesión en [duckdns.org](https://www.duckdns.org/).
2. En **sub domain** escribir `servertaller3` y pulsar **add domain**.
3. En **current ip** poner `137.131.240.182` y pulsar **update ip**.

Resultado: `servertaller3.duckdns.org` apunta a la VM.

![Dominio configurado en DuckDNS](imagenes/07-duckdns.png)

---

## 8. Comprobación final

Abrir en un navegador (idealmente desde el celular con datos móviles, fuera de la red de la VM):

```
http://servertaller3.duckdns.org
```

![Sitio funcionando](imagenes/08-sitio-publicado.png)

---

## Resultado

 Sitio en línea: http://servertaller3.duckdns.org

## Problemas encontrados y soluciones

| Problema | Causa | Solución |
|---|---|---|
| `Permission denied (publickey)` | Usuario o llave incorrectos | Usar `opc` y la llave correcta (`llave-web1.key`) |
| `sudo: No such file or directory` | Se ejecutó el comando en Cloud Shell, no en la VM | Conectarse primero por SSH |
| `syntax error near unexpected token` | Código JS pegado en la terminal en vez de en `nano` | Abrir `nano` y pegar ahí |
| Archivo `index.htmlno` en la carpeta | Error de tipeo al guardar con `nano` | Borrarlo con `sudo rm /var/www/misitio/index.htmlno` |
| Nginx sin acceso a los archivos | SELinux / permisos | `chown`, `chmod` y `setenforce 0` |

## Conclusiones

- Se desplegó una VM en OCI y se configuró Nginx para servir un sitio propio desde `/var/www/misitio`.
- Se abrió el puerto 80 y se asoció un dominio gratuito con DuckDNS.
- Los errores del proceso (usuario, llave, SELinux) ayudaron a entender mejor cómo funciona el acceso y la seguridad en Linux.
