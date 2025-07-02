# SICAM Standards - PHP 5.6 + MySQL + jQuery + Bootstrap

> Estándar de desarrollo para ERP SICAM  
> Mantener stack, mejorar estructura, asegurar consistencia.

---

## ⚙️ CONTEXTO TÉCNICO

- PHP 5.6  
- MySQL (con `mysqli_*` exclusivamente)  
- Paradigma funcional / imperativo  
- Charset: UTF-8 (`utf8mb4`)  
- jQuery + Bootstrap  
- Sin frameworks

---

## 📁 ESTRUCTURA MODULAR

| Carpeta / Archivo                    | Descripción                       |
|-------------------------------------|-----------------------------------|
| /tabla_usuario/                     | Módulo ejemplo base               |
| ├── tabla.php                       | Frontend                          |
| ├── /server/controlador_tabla.php   | Controlador MySQL                 |
| └── /inc/conexion.php               | Configuración de base de datos    |

### 🧪 Ejemplo aplicado (módulo real):

| Carpeta / Archivo                    | Descripción                       |
|-------------------------------------|-----------------------------------|
| /ejecutivo/                         | Módulo real de citas              |
| ├── citas.php                       | Frontend                          |
| ├── /server/controlador_citas.php   | Controlador de citas              |
| └── /inc/conexion.php               | Configuración BD                  |

> ⚠️ **Nota:** Todos los archivos van en **plural** (ej: `controlador_citas.php`)

---

## 🔌 CONEXIÓN Y CONFIGURACIÓN BASE

### Archivo: `/inc/conexion.php`

```php
<?php
	$host = "localhost";
	$user = "sicam_user"; 
	$pass = "sicam_pass";
	$database = "sicam_db";
	$connection = mysqli_connect($host, $user, $pass, $database);

	if (!$connection) {
		die('Error de conexión: ' . mysqli_connect_error());
	}

	mysqli_set_charset($connection, "utf8mb4");
?>
```

---

## 🔁 FUNCIONES BÁSICAS

```php
<?php
// Función para ejecutar consultas y obtener datos
function ejecutarConsulta($query, $connection) {
	$result = mysqli_query($connection, $query);
	if (!$result) return false;
	$datos = [];
	while($row = mysqli_fetch_assoc($result)) {
		$datos[] = $row;
	}
	return $datos;
}

// Función para escape de datos (prevención SQL Injection)
function escape($valor, $connection) {
	return mysqli_real_escape_string($connection, $valor);
}

// Respuesta exitosa estándar
function respuestaExito($data = null, $message = 'OK') {
	return json_encode([
		'success' => true,
		'data' => $data,
		'message' => $message
	], JSON_UNESCAPED_UNICODE);
}

// Respuesta de error estándar
function respuestaError($message = 'Error', $code = 400) {
	return json_encode([
		'success' => false,
		'message' => $message,
		'code' => $code
	], JSON_UNESCAPED_UNICODE);
}
?>
```

---

## 🔄 INTEGRACIÓN AJAX (Frontend jQuery)

```javascript
// Llamada AJAX estándar
$.ajax({
	url: 'server/controlador_citas.php',
	type: 'POST',
	data: {
		action: 'obtener_citas',
		filtro: 'algún_valor'
	},
	dataType: 'json',
	success: function(response) {
		if(response.success) {
			renderizarCitas(response.data);
		} else {
			alert('Error: ' + response.message);
		}
	}
});

// Función para renderizar datos
function renderizarCitas(citas) {
	var html = '';
	citas.forEach(function(cita) {
		html += '<div>' + cita.nom_cit + ' - ' + cita.tel_cit + ' (Ejecutivo: ' + cita.nom_eje + ')</div>';
	});
	$('#contenedor-citas').html(html);
}
```

---

## 🔧 BACKEND - CONTROLADOR

### Archivo: `/server/controlador_citas.php`

```php
<?php
include '../inc/conexion.php';
header('Content-Type: application/json; charset=utf-8');

if ($_SERVER['REQUEST_METHOD'] == 'POST') {
	
	$action = escape($_POST['action'], $connection);
	
	switch($action) {

		case 'obtener_citas':
			$filtro = isset($_POST['filtro']) ? escape($_POST['filtro'], $connection) : '';
			$query = "SELECT c.id_cit, c.nom_cit, c.tel_cit, e.nom_eje 
					 FROM cita c
					 LEFT JOIN ejecutivo e ON c.id_eje2 = e.id_eje
					 WHERE c.nom_cit LIKE '%$filtro%'
					 ORDER BY c.nom_cit ASC";

			$datos = ejecutarConsulta($query, $connection);

			if($datos !== false) {
				echo respuestaExito($datos, 'Citas obtenidas correctamente');
			} else {
				echo respuestaError('Error al consultar citas: ' . mysqli_error($connection) . ' Query: ' . $query);
			}
		break;

		case 'guardar_cita':
			$nom_cit = escape($_POST['nom_cit'], $connection);
			$tel_cit = escape($_POST['tel_cit'], $connection);
			$id_eje2 = escape($_POST['id_eje2'], $connection);

			$query = "INSERT INTO cita (nom_cit, tel_cit, id_eje2) 
					 VALUES ('$nom_cit', '$tel_cit', '$id_eje2')";

			if(mysqli_query($connection, $query)) {
				echo respuestaExito(['id' => mysqli_insert_id($connection)], 'Cita guardada correctamente');
			} else {
				echo respuestaError('Error al guardar cita: ' . mysqli_error($connection) . ' Query: ' . $query);
			}
		break;

		default:
			echo respuestaError('Acción no válida');
		break;
	}

	mysqli_close($connection);
	exit;
}
?>
```

---

## 🖼️ EJEMPLO BÁSICO EN FRONTEND

### Archivo: `citas.php`

```html
<div id="contenedor-citas"></div>

<script>
$.ajax({
	url: 'server/controlador_citas.php',
	type: 'POST',
	data: { action: 'obtener_citas' },
	dataType: 'json',
	success: function(response) {
		if(response.success) {
			renderizar(response.data);
		}
	}
});

function renderizar(citas) {
	var html = '';
	citas.forEach(function(c) {
		html += '<div>' + c.nom_cit + ' - ' + c.tel_cit + '</div>';
	});
	$('#contenedor-citas').html(html);
}
</script>
```

---

## 🧱 TEMPLATE RENDERER (PHP + HTML)

```php
<?php
$datos = json_decode($_POST['datos'], true);
foreach($datos as $item):
?>
	<div><?= $item['nom_cit'] ?> - <?= $item['tel_cit'] ?></div>
<?php endforeach; ?>
```

---

## 📌 NOTAS IMPORTANTES

- ✅ Header `Content-Type` con `charset=utf-8` en controladores
- ✅ Controlador devuelve JSON con array estructurado
- ✅ `response.data` es un array de objetos
- ✅ `escape()` obligatorio para prevenir SQL Injection
- ✅ Funciones pequeñas (máximo 20-30 líneas)
- ✅ Separar lógica de presentación
- ✅ Mostrar query en errores para debugging
- ✅ Vibecodear moderadamente 🚀

---

> *"Por cada paso de éxito, subir 2 de humildad"* - **ericorps**
