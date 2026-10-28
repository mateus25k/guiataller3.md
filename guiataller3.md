# Despliegue de un sitio web en Oracle Cloud (OCI) con Nginx y DuckDNS

Práctica de infraestructura: montar una máquina virtual en Oracle Cloud, instalar un servidor web y publicar una página accesible desde internet con un dominio gratuito.

🌐 **Sitio publicado:** http://miwebsi.duckdns.org

---

## Resumen

| Elemento | Valor |
|---|---|
| Proveedor | Oracle Cloud Infrastructure (región Brazil East - São Paulo) |
| Sistema operativo | Oracle Linux 9 |
| Nombre de la VM | `mi-web` |
| Red | VCN `red-infraestructura` / subred pública `subred-web` |
| IP pública | `163.176.111.62` |
| IP privada | `10.0.0.244` |
| Servidor web | Nginx |
| Dominio | `miwebsi.duckdns.org` (DuckDNS) |

---

## 1. Crear la máquina virtual

1. En la consola de OCI ir a **Compute → Instances → Create instance**.
2. Nombre: `mi-web`.
3. Imagen: **Oracle Linux 9**.
4. Red: reutilizar la VCN `red-infraestructura` y la subred pública `subred-web`, con IP pública asignada.
5. Generar o cargar la llave SSH (queda guardada como `llave-web.key`) y pulsar **Create**.

Una vez creada, en la pestaña **Networking** se ven la IP pública, la IP privada, la subred y el hostname:

![Datos de red de la instancia](imagenes/01-red-instancia.png)

---

## 2. Conectarse por SSH desde Cloud Shell

1. Abrir **Cloud Shell** en la consola de OCI.
2. Subir la llave privada con el menú **Upload** y darle permisos seguros:

```bash
chmod 600 ~/llave-web.key
```

3. Conectarse (en Oracle Linux el usuario es `opc`):

```bash
ssh -i ~/llave-web.key opc@163.176.111.62
```

4. Responder `yes` cuando pregunte si se confía en el host.

![Conexión SSH desde Cloud Shell](imagenes/02-ssh-cloudshell.png)

---

## 3. Instalar Nginx y Git

```bash
sudo dnf install -y nginx git
```

![Instalación de nginx y git en curso](imagenes/03-instalacion-nginx.png)

Al terminar debe aparecer el mensaje **Complete!**:

![Instalación completada](imagenes/04-instalacion-completa.png)

---

## 4. Crear la página web

Editar el archivo principal que sirve Nginx:

```bash
sudo nano /usr/share/nginx/html/index.html
```

Estructura de la página: un título "Mi sitio", un menú (Inicio, Servicios, Contacto), una sección por cada apartado con su imagen y un botón **Saludar** que muestra un mensaje con JavaScript.

```html
<script>
  function saludar() {
    document.getElementById("mensaje").textContent = "¡Hola! Bienvenido.";
  }
</script>
```

![Código de la página en el servidor](imagenes/05-codigo-html.png)

---

## 5. Iniciar Nginx

```bash
sudo systemctl enable --now nginx
```

---

## 6. Verificar el servicio y abrir el puerto 80 en el firewall de la VM

```bash
systemctl is-active nginx
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

Debe responder `active` y luego `success` en los dos comandos de firewall:

![Nginx activo y firewall configurado](imagenes/06-nginx-firewall.png)

> **Nota:** además del firewall de la VM, la subred de OCI debe tener una regla de entrada (Ingress) para el puerto 80 en su Security List.

---

## 7. Probar con la IP pública

Abrir en el navegador:

```
http://163.176.111.62
```

---

## 8. Configurar el subdominio en DuckDNS

1. Iniciar sesión en [duckdns.org](https://www.duckdns.org/).
2. En **sub domain** escribir `miwebsi` y pulsar **add domain**.
3. En **current ip** poner la IP pública de la instancia (`163.176.111.62`) y pulsar **update ip**.
4. Resultado: `miwebsi.duckdns.org` apunta a la VM.

![Dominio configurado en DuckDNS](imagenes/07-duckdns.png)

---

## 9. Comprobación desde fuera de la red de la VM

Desde un celular con datos móviles (fuera de la red de la VM) abrir:

```
http://miwebsi.duckdns.org
```

![Sitio funcionando con el dominio](imagenes/08-sitio-publicado.png)

---

## Resultado

 Sitio web en línea: http://miwebsi.duckdns.org

## Conclusiones

- Se desplegó una VM en OCI con Oracle Linux 9 dentro de una red ya existente.
- Nginx quedó instalado, habilitado y sirviendo contenido por el puerto 80.
- Con DuckDNS se asignó un nombre de dominio gratuito a la IP pública, sin comprar dominio.
