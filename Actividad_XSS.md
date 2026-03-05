Lo primero es dirigirme a la ubicación en la que en el ejercicio anterior cree toda la infraestructura con la herramienta docker. En mi caso en ~/Unidad3/CreacionEntornoPruebas/docker-compose-lamp. Una vez aquí me iré al directorio /www

```bash
cd www
```

Una vez ya estoy en la ruta correcta creo un  fichero llamado comment.php qeu es donde pondre el código vulnerable.

```bash
nano comment.php
```

![Ubicación y crear archivo](/imagenes/img1.png)

Y dentro pondré lo siguiente: 

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
Como se puede ver a continuación

![Contenido vulnerabilidad](/imagenes/img2.png)
---

A continuación en un buscador cualquiera de la MV, buscaré la siguiente url http://localhost/comment.php y pongo un comentario cualquiera

![Contenido vulnerabilidad](/imagenes/img3.png)

---

### Explotaciones

**Explotación 1**

Ahora comenzaré con la explotación del formulario, para ello pondre el siguiente "comentario"

> <script>alert('XSS ejecutado!')</script>

Aparece un mensaje de alerta en el navegador que significa que la aplicación es vulnerable

![Explotación 1](/imagenes/img4.png)

---

**Explotación 2**

Pondré a que se redirija el tráfico a una página de phishing:

> <script>window.location='https://fakeupdate.net/win11/'</script>
![Comentario](/imagenes/img5.png)
Esta sería la página que se abre según envío el exploit.
![Redirección](/imagenes/img6.png)
