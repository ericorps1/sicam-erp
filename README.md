# SICAM Standards - PHP 5.6 + MySQL + jQuery + Bootstrap

> 🧠 Estándar de desarrollo para ERP SICAM  
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

## 📊 HANDSONTABLE PARA CRUDS

ERP SICAM utiliza **Handsontable** para casi todos los CRUD de los módulos.

### 🔗 Referencia oficial:
- **Documentación:** https://handsontable.com/demo
- **Incorporación:** Manual (descarga de archivos CSS/JS)
- **Uso:** Tablas editables con integración AJAX

### 💡 Implementación básica:
```html
<!-- Dependencias manuales -->
<link rel="stylesheet" href="handsontable/handsontable.full.min.css">
<script src="handsontable/handsontable.full.min.js"></script>

<!-- Contenedor -->
<div id="tabla-citas"></div>

<script>
// Inicialización de Handsontable para citas
var hot = new Handsontable(document.getElementById('tabla-citas'), {
	data: datos,  // Array bidimensional con los datos de citas
	colHeaders: ['ID', 'NOMBRE', 'TELÉFONO', 'EJECUTIVO'],  // Títulos de columnas de cita
	columns: [
		{ type: 'text', readOnly: true },  // ID de cita (solo lectura)
		{ type: 'text' },  // Nombre de la cita
		{ type: 'text' },  // Teléfono de la cita
		{ type: 'dropdown', source: ['Ejecutivo 1', 'Ejecutivo 2'] }  // Dropdown de ejecutivos
	],
	// Evento que se dispara cuando cambia una celda
	afterChange: function(changes, source) {
		// Evitar procesar cambios cuando se carga data inicial
		if (changes && source !== 'loadData') {
			// Recorrer todos los cambios realizados
			changes.forEach(([row, prop, oldValue, newValue]) => {
				// Llamar función para guardar cada cambio
				guardarCambio(row, prop, newValue);
			});
		}
	}
});

// Función para guardar cambios via AJAX
function guardarCambio(row, column, value) {
	// Patrón Column-to-Field Mapping para mapear índice de columna a campo BD
	var campo = obtenerCampo(column);  // Obtener nombre del campo de BD
	var id = hot.getDataAtCell(row, 0);  // Obtener ID de la cita (columna 0)
	
	// Llamada AJAX para guardar cambio en cita
	$.ajax({
		url: 'server/controlador_citas.php',  // Endpoint del controlador de citas
		type: 'POST',  // Método HTTP
		data: {
			action: 'actualizar_cita',  // Acción a realizar
			campo: campo,  // Campo de BD a actualizar
			valor: value,  // Nuevo valor
			id_cit: id  // ID de la cita
		},
		dataType: 'json',  // Esperamos respuesta JSON
		success: function(response) {
			// Manejar respuesta exitosa
			if(response.success) {
				console.log('Cita actualizada');
			} else {
				alert('Error: ' + response.message);
			}
		}
	});
}

// Función para mapear columna a campo de BD (implementar según necesidades)
function obtenerCampo(column) {
	// Mapeo de índice de columna a nombre de campo en tabla cita
	var campos = {
		0: 'id_cit',  // Columna 0 = campo id_cit
		1: 'nom_cit',  // Columna 1 = campo nom_cit (nombre)
		2: 'tel_cit',  // Columna 2 = campo tel_cit (teléfono)
		3: 'id_eje2'   // Columna 3 = campo id_eje2 (ejecutivo)
	};
	return campos[column];  // Retornar el campo correspondiente
}
</script>
```

### 🔧 Backend para Handsontable:

```php
<?php
// Archivo: server/controlador_citas.php
include '../inc/conexion.php';
header('Content-Type: application/json; charset=utf-8');

if ($_SERVER['REQUEST_METHOD'] == 'POST') {
	
	$action = escape($_POST['action'], $connection);
	
	switch($action) {
		
		case 'actualizar_cita':
			$campo = escape($_POST['campo'], $connection);
			$valor = escape($_POST['valor'], $connection);
			$id_cit = escape($_POST['id_cit'], $connection);
			
			// Patrón Field-Value dinámico para UPDATE
			$query = "UPDATE cita SET $campo = '$valor' WHERE id_cit = '$id_cit'";
			
			if(mysqli_query($connection, $query)) {
				echo respuestaExito(null, 'Campo actualizado correctamente');
			} else {
				echo respuestaError('Error al actualizar: ' . mysqli_error($connection) . ' Query: ' . $query);
			}
		break;
		
		// Otros casos del controlador...
	}
	
	mysqli_close($connection);
	exit;
}
?>
```

> **Nota:** La implementación específica y mapeo de campos queda a criterio del desarrollador según las necesidades del módulo.

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
