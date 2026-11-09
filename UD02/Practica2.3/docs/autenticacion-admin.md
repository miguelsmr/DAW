# Práctica 2.3 — Autenticación básica de `admin`

**Alumno:** Miguel Luzón Prado · **Módulo:** Despliegue de Aplicaciones Web (2º DAW) · **Proyecto:** TaskFlow

**Objetivo:** proteger una zona del sitio con autenticación HTTP básica.

**Entorno:** Ubuntu Server 26.10 en VMware Fusion (`taskflowserver`), Apache 2.4.68, host virtual `taskflow.local` de la práctica 2.2.

---

## 1. Autenticación HTTP básica

1. El navegador pide `/admin/` y Apache responde **`401 Unauthorized`** con la cabecera `WWW-Authenticate: Basic realm="..."`.
2. El navegador muestra la ventana de usuario y contraseña.
3. El navegador repite la petición con la cabecera `Authorization: Basic <credenciales>`, donde las credenciales son `usuario:contraseña` codificadas en **Base64**. Ejemplo con datos inventados:

   ```
   admin:EjemploClave  →  YWRtaW46RWplbXBsb0NsYXZl
   Authorization: Basic YWRtaW46RWplbXBsb0NsYXZl
   ```
4. Apache comprueba el usuario y la contraseña contra `.htpasswd`: si coinciden responde **`200 OK`**; si no, otra vez `401`.

**Por qué no es segura sin HTTPS:** Base64 es una **codificación, no un cifrado**; cualquiera que capture el tráfico la decodifica al instante (`echo YWRtaW46RWplbXBsb0NsYXZl | base64 -d`). Además, las credenciales se envían en **cada** petición. Por eso solo debe usarse sobre **HTTPS**, que cifra toda la comunicación.

**Alternativas:**

| Método | Cómo funciona |
|---|---|
| Digest (`AuthType Digest`) | Envía un hash de la contraseña en lugar de la contraseña; mejor que Basic sin HTTPS, pero usa MD5 y está en desuso |
| Formulario + sesión | Login en una página propia; el servidor crea una sesión y el navegador guarda una cookie |
| JWT (JSON Web Token) | Tras el login, el servidor entrega un token firmado que el cliente envía en `Authorization: Bearer ...`; habitual en APIs como la de TaskFlow |

---

## 2. Crear `/admin` dentro del DocumentRoot

```bash
sudo mkdir -p /var/www/taskflow/dist/admin
sudo tee /var/www/taskflow/dist/admin/index.html > /dev/null <<'HTML'
<!DOCTYPE html>
<html lang="es">
<head><meta charset="UTF-8"><title>TaskFlow - Admin</title></head>
<body>
  <h1>TaskFlow - Zona de administración</h1>
  <p>Acceso permitido: usuario autenticado.</p>
</body>
</html>
HTML
```

---

## 3. Fichero de contraseñas `.htpasswd`

```bash
sudo htpasswd -c /etc/apache2/.htpasswd admin
sudo chown root:www-data /etc/apache2/.htpasswd
sudo chmod 640 /etc/apache2/.htpasswd
```

- `htpasswd` (paquete `apache2-utils`) crea y gestiona ficheros de usuarios de Apache. Pide la contraseña dos veces sin mostrarla y guarda un **hash** (`$apr1$...`, MD5 de Apache), nunca la contraseña en claro.
- **`-c` (create)** crea el fichero. **Solo se usa la primera vez**: si se usa con un fichero que ya existe, lo sobrescribe y se borran los demás usuarios.
- Está en `/etc/apache2`, **fuera del DocumentRoot**, para que no se pueda descargar desde la web.
- Permisos `640` y grupo `www-data`: solo root puede modificarlo y Apache (`www-data`) solo leerlo.

![htpasswd](capturas/01-htpasswd.png)

### Política de la contraseña

La contraseña de `admin` tiene **al menos 12 caracteres** y combina **mayúsculas, minúsculas, números y símbolos**, sin palabras del diccionario ni datos personales. La longitud y la variedad de caracteres hacen inviable adivinarla por fuerza bruta o por diccionario, y es la cuenta que da acceso a la administración. No se escribe en ningún documento ni en el historial (el `curl` de la verificación la pide por teclado).

**Rotación:** se cambia periódicamente (por ejemplo, cada 90 días) o de inmediato si se sospecha que se ha filtrado, ejecutando `htpasswd` **sin `-c`** sobre el mismo usuario:

```bash
sudo htpasswd /etc/apache2/.htpasswd admin
```

### Añadir y eliminar usuarios

```bash
sudo htpasswd /etc/apache2/.htpasswd usuario2      # añadir (sin -c)
sudo htpasswd -D /etc/apache2/.htpasswd usuario2   # eliminar
sudo cat /etc/apache2/.htpasswd                    # ver usuarios (con su hash)
```

No hace falta reiniciar Apache: lee el fichero en cada petición.

---

## 4. `<Directory>` en `taskflow.conf`

```apache
# Host por defecto (primero del puerto 80): atiende lo que no sea taskflow.local, p. ej. localhost
<VirtualHost *:80>
    ServerName localhost
    DocumentRoot /var/www/html
    <Location />
        Require all denied
    </Location>
    ErrorLog ${APACHE_LOG_DIR}/default-error.log
    CustomLog ${APACHE_LOG_DIR}/default-access.log combined
</VirtualHost>

# Sitio de TaskFlow
<VirtualHost *:80>
    ServerName taskflow.local
    DocumentRoot /var/www/taskflow/dist

    # Zona de administración protegida con autenticación básica
    <Directory /var/www/taskflow/dist/admin>
        AuthType Basic
        AuthName "TaskFlow - Zona de administracion"
        AuthUserFile /etc/apache2/.htpasswd
        Require valid-user
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/taskflow-error.log
    CustomLog ${APACHE_LOG_DIR}/taskflow-access.log combined
</VirtualHost>
```

| Directiva | Función |
|---|---|
| `<Directory /var/www/taskflow/dist/admin>` | Aplica las reglas a la carpeta `/admin` y todo lo que contiene |
| `AuthType Basic` | Tipo de autenticación: HTTP básica |
| `AuthName` | Texto del *realm* que se envía en `WWW-Authenticate` |
| `AuthUserFile` | Fichero con los usuarios y contraseñas |
| `Require valid-user` | Solo entra un usuario del fichero con la contraseña correcta |

Usa los módulos `auth_basic`, `authn_file` y `authz_user`, que Apache trae activos por defecto (se ven en `modulos-antes.txt` de la práctica 2.1).

```bash
sudo apachectl configtest
sudo systemctl reload apache2
```

![taskflow.conf](capturas/02-taskflow-conf.png)

---

## 5. Verificación

### Con `curl`

```bash
curl -I http://taskflow.local/admin/                     # sin credenciales
curl -I -u admin:incorrecta http://taskflow.local/admin/ # contraseña incorrecta
curl -I -u admin http://taskflow.local/admin/            # pide la contraseña correcta
```

![curl 401 y 200](capturas/03-curl-401-200.png)

| Petición | Resultado |
|---|---|
| Sin credenciales | **`401 Unauthorized`** + `WWW-Authenticate: Basic realm="TaskFlow - Zona de administracion"` |
| Contraseña incorrecta | **`401 Unauthorized`** |
| Credenciales correctas | **`200 OK`** |

### Con el navegador

Sin credenciales, el navegador muestra la ventana de inicio de sesión (y avisa de que la conexión no es privada, porque va por HTTP):

![Ventana de login](capturas/04-navegador-login.png)

Con `admin` y la contraseña correcta, se muestra la zona de administración:

![Zona de administración](capturas/05-navegador-admin.png)

---

## 6. Criterios de aceptación

- [x] `/admin` devuelve **401** sin credenciales.
- [x] Con credenciales correctas permite el acceso (**200**).
- [x] La contraseña del usuario `admin` es segura (política del apartado 3).
