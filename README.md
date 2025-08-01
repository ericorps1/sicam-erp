# SICAM Standards - PHP 5.6 + MySQL + jQuery + Bootstrap

> 🧠 Estándar de desarrollo para ERP SICAM  
> Mantener stack, mejorar estructura, asegurar consistencia.

---

## 📖 TEMPLATE DE SESIÓN SICAM

**Ver documentación completa del sistema de sesiones:** [ERP - SICAM Template de Sesión](https://github.com/ericorps1/template_sicam)

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
# 🔌 WebSocket para SICAM - Guía Completa

**Servidor WebSocket disponible:** `wss://socket.ahjende.com/wss/?encoding=text`

## 🎯 **¿Qué es WebSocket?**

Imagina un **teléfono** entre todos los navegadores conectados. Cuando alguien habla, todos escuchan **instantáneamente**.

```
                Usuario C
                    ↑
    Usuario A ← WebSocket → Usuario B  
                    ↓
                Usuario D
```

**Ventaja vs AJAX**: No necesitas preguntar "¿hay algo nuevo?" cada segundo. El servidor te avisa automáticamente.

---

## 🏗️ **Las 4 Funcionalidades Básicas**

### 1. **`onopen`** - Conexión establecida
```javascript
socket.onopen = () => console.log('✅ Conectado');
```

### 2. **`onmessage`** - Recibir datos
```javascript
socket.onmessage = (event) => console.log('📨 Recibido:', event.data);
```

### 3. **`onclose`** - Conexión cerrada
```javascript
socket.onclose = () => console.log('🔴 Desconectado');
```

### 4. **`onerror`** - Error en conexión
```javascript
socket.onerror = (error) => console.error('❌ Error:', error);
```

### **`send()`** - Enviar datos
```javascript
socket.send(JSON.stringify({ mensaje: 'Hola' }));
```

---

## 🚀 **Implementación SICAM**

### **Servidor WebSocket ya existe:**
```
wss://socket.ahjende.com/wss/?encoding=text
```

**Solo necesitas frontend JavaScript.** El servidor está listo.

---

## 🧪 **Ejemplo 1: Prueba Básica**

**¿Qué hace?** Conecta al WebSocket y permite enviar mensajes de prueba. Al hacer clic en "Enviar", todas las pestañas abiertas ven el mismo mensaje.

```html
<!DOCTYPE html>
<html>
<body>
    <h1>Test WebSocket Simple</h1>
    <button onclick="enviar()">Enviar</button>
    <div id="log"></div>

    <script>
        const socket = new WebSocket('wss://socket.ahjende.com/wss/?encoding=text');
        
        socket.onopen = () => {
            document.getElementById('log').innerHTML += '<p>✅ Conectado</p>';
        };
        
        socket.onmessage = (event) => {
            document.getElementById('log').innerHTML += '<p>📨 ' + event.data + '</p>';
        };
        
        function enviar() {
            socket.send(JSON.stringify({
                mensaje: 'Hola desde ' + Math.random()
            }));
        }
    </script>
</body>
</html>
```

**Prueba**: Abre en 2 pestañas, haz clic en una, ve el mensaje en ambas.

---

## 🎯 **Ejemplo 2: WebSocket + Handsontable**

**¿Qué hace?** Simula el sistema de citas real. Cuando editas una celda en una pestaña, el cambio aparece automáticamente en todas las demás pestañas abiertas. Incluye logs para ver exactamente qué mensajes se envían y reciben.

```html
<!DOCTYPE html>
<html>
<head>
    <title>WebSocket + Handsontable SICAM</title>
    <!-- Handsontable CDN -->
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/handsontable@14.5.0/dist/handsontable.full.min.css">
    <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/handsontable@14.5.0/dist/handsontable.full.min.js"></script>
</head>
<body>
    <h1>Test WebSocket + Handsontable</h1>
    <div id="tabla" style="width: 600px; height: 300px;"></div>
    <p>Mi ID Ejecutivo: <input id="miId" value="1" style="width: 50px;"> (Cambia a 2 en otra pestaña)</p>
    <div id="log" style="margin-top: 20px; padding: 10px; border: 1px solid #ccc; height: 200px; overflow-y: scroll;"></div>

    <script>
        // WebSocket
        const socket = new WebSocket('wss://socket.ahjende.com/wss/?encoding=text');
        
        function log(mensaje) {
            const time = new Date().toLocaleTimeString();
            document.getElementById('log').innerHTML += `<p>[${time}] ${mensaje}</p>`;
            document.getElementById('log').scrollTop = document.getElementById('log').scrollHeight;
        }
        
        // Datos de prueba (simular citas)
        const datos = [
            [1, 'Juan Pérez', '555-1234'],
            [2, 'María García', '555-5678'],
            [3, 'Carlos López', '555-9999']
        ];
        
        // Handsontable
        const hot = new Handsontable(document.getElementById('tabla'), {
            data: datos,
            colHeaders: ['ID Cita', 'Nombre Cliente', 'Teléfono'],
            columns: [
                {readOnly: true}, // ID no editable
                {type: 'text'},   // Nombre
                {type: 'text'}    // Teléfono
            ],
            licenseKey: 'non-commercial-and-evaluation',
            
            afterChange: function(changes, source) {
                log(`📝 afterChange disparado con source: ${source}`);
                
                if (source !== 'loadData' && source !== 'websocket') {
                    changes.forEach(function([row, col, oldValue, newValue]) {
                        
                        const id_cit = hot.getDataAtCell(row, 0);
                        const campo = col === 1 ? 'nom_cit' : 'tel_cit';
                        const miId = parseInt(document.getElementById('miId').value);
                        
                        // Enviar por WebSocket
                        const mensaje = {
                            tipo: 'cita_actualizada',
                            id_cit: id_cit,
                            campo: campo,
                            valor: newValue,
                            id_ejecutivo: miId
                        };
                        
                        socket.send(JSON.stringify(mensaje));
                        log(`📤 ENVIADO: Cita ${id_cit}, campo ${campo} = "${newValue}"`);
                    });
                }
            }
        });
        
        // WebSocket eventos
        socket.onopen = function() {
            log('✅ WebSocket conectado');
        };
        
        socket.onmessage = function(event) {
            const mensaje = JSON.parse(event.data);
            const miId = parseInt(document.getElementById('miId').value);
            
            log(`📨 RECIBIDO: ${JSON.stringify(mensaje)}`);
            
            // Ignorar mis propios mensajes
            if (mensaje.id_ejecutivo === miId) {
                log('⏭️ Ignorando mi propio mensaje');
                return;
            }
            
            // Solo procesar actualizaciones de citas
            if (mensaje.tipo === 'cita_actualizada') {
                // Buscar fila y actualizar
                const filas = hot.getData();
                for (let i = 0; i < filas.length; i++) {
                    if (filas[i][0] == mensaje.id_cit) {
                        const col = mensaje.campo === 'nom_cit' ? 1 : 2;
                        
                        log(`🔄 Actualizando fila ${i}, columna ${col} con "${mensaje.valor}"`);
                        
                        // Actualizar con source 'websocket' (IMPORTANTE)
                        hot.setDataAtCell(i, col, mensaje.valor, 'websocket');
                        
                        // Resaltar visualmente
                        const celda = hot.getCell(i, col);
                        if (celda) {
                            celda.style.backgroundColor = '#ffeb3b';
                            celda.style.transition = 'background-color 0.3s';
                            setTimeout(() => {
                                celda.style.backgroundColor = '';
                            }, 2000);
                        }
                        
                        break;
                    }
                }
            }
        };
        
        socket.onclose = function() {
            log('🔴 WebSocket desconectado');
        };
        
        socket.onerror = function(error) {
            log('❌ Error WebSocket: ' + error);
        };
    </script>
</body>
</html>
```

**Prueba**: Abre en 2 pestañas, cambia ID ejecutivo en cada una, edita celdas y ve actualizaciones en tiempo real.

---

## 🔄 **Evitar Bucles Infinitos**

### **Problema:**
```
Usuario A cambia → Envía WebSocket → Usuario A recibe → Cambia tabla → Envía otra vez → BUCLE
```

### **🚨 Ejemplo de Bucle (¿Qué NO hacer?)**

Si en el Ejemplo 2 comentas estas líneas:

```javascript
// Comentar estas líneas causa BUCLE INFINITO
// if (mensaje.id_ejecutivo === miId) {
//     log('⏭️ Ignorando mi propio mensaje');
//     return;
// }
```

**Resultado:** El usuario que edita recibirá su propio mensaje, actualizará la tabla, disparará `afterChange`, enviará otro WebSocket, y así infinitamente.

### **🧪 Prueba de Bucle Infinito**

```html
<!DOCTYPE html>
<html>
<head>
    <title>Test WebSocket BUCLE INFINITO - NO USAR</title>
    <!-- Handsontable CDN -->
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/handsontable@14.5.0/dist/handsontable.full.min.css">
    <script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/handsontable@14.5.0/dist/handsontable.full.min.js"></script>
</head>
<body>
    <h1>⚠️ Test BUCLE INFINITO - Solo para Demostración</h1>
    <div id="tabla" style="width: 600px; height: 300px;"></div>
    <p style="color: red;"><strong>ADVERTENCIA:</strong> Este ejemplo genera bucle infinito. Refresca la página para parar.</p>
    <div id="log" style="margin-top: 20px; padding: 10px; border: 1px solid #ccc; height: 200px; overflow-y: scroll;"></div>

    <script>
        const socket = new WebSocket('wss://socket.ahjende.com/wss/?encoding=text');
        
        function log(mensaje) {
            const time = new Date().toLocaleTimeString();
            document.getElementById('log').innerHTML += `<p>[${time}] ${mensaje}</p>`;
            document.getElementById('log').scrollTop = document.getElementById('log').scrollHeight;
        }
        
        const datos = [
            [1, 'Juan Pérez', '555-1234'],
            [2, 'María García', '555-5678']
        ];
        
        const hot = new Handsontable(document.getElementById('tabla'), {
            data: datos,
            colHeaders: ['ID Cita', 'Nombre Cliente', 'Teléfono'],
            columns: [
                {readOnly: true}, // ID no editable
                {type: 'text'},   // Nombre
                {type: 'text'}    // Teléfono
            ],
            licenseKey: 'non-commercial-and-evaluation',
            
            afterChange: function(changes, source) {
                log(`📝 afterChange disparado con source: ${source}`);
                
                // ❌ SIN VALIDACIÓN DE SOURCE - ESTO CAUSA EL BUCLE
                // Removemos la validación source !== 'websocket'
                if (source !== 'loadData') {
                    changes.forEach(function([row, col, oldValue, newValue]) {
                        const id_cit = hot.getDataAtCell(row, 0);
                        const campo = col === 1 ? 'nom_cit' : 'tel_cit';
                        
                        const mensaje = {
                            tipo: 'cita_actualizada',
                            id_cit: id_cit,
                            campo: campo,
                            valor: newValue,
                            id_ejecutivo: 1
                        };
                        
                        socket.send(JSON.stringify(mensaje));
                        log(`📤 ENVIADO: ${JSON.stringify(mensaje)}`);
                    });
                }
            }
        });
        
        socket.onopen = function() {
            log('✅ WebSocket conectado');
        };
        
        socket.onmessage = function(event) {
            const mensaje = JSON.parse(event.data);
            log(`📨 RECIBIDO: ${JSON.stringify(mensaje)}`);
            
            // ❌ SIN VALIDACIÓN DE USUARIO - Procesamos TODOS los mensajes
            
            if (mensaje.tipo === 'cita_actualizada') {
                const filas = hot.getData();
                for (let i = 0; i < filas.length; i++) {
                    if (filas[i][0] == mensaje.id_cit) {
                        const col = mensaje.campo === 'nom_cit' ? 1 : 2;
                        
                        log(`🔄 Actualizando fila ${i}, columna ${col} con "${mensaje.valor}"`);
                        
                        // ❌ SIN source 'websocket' - Esto dispara afterChange otra vez
                        hot.setDataAtCell(i, col, mensaje.valor);
                        
                        break;
                    }
                }
            }
        };
        
        socket.onclose = function() {
            log('🔴 WebSocket desconectado');
        };
    </script>
</body>
</html>
```

**¿Qué pasa ahora?** Al editar una celda:
1. `afterChange` (source='edit') → envía WebSocket
2. `onmessage` → recibe su propio mensaje  
3. `setDataAtCell` SIN source → dispara `afterChange` (source='edit') otra vez
4. **BUCLE INFINITO** 🔄

**Los 2 cambios clave para el bucle:**
- Quitar `&& source !== 'websocket'` del `afterChange`
- Quitar el tercer parámetro de `setDataAtCell`

### **Solución 1: Por ID de Ejecutivo**
```javascript
// Al enviar
socket.send(JSON.stringify({
    id_ejecutivo: miId, // Quién envía
    // ... otros datos
}));

// Al recibir
socket.onmessage = function(event) {
    const mensaje = JSON.parse(event.data);
    
    if (mensaje.id_ejecutivo === miId) {
        return; // Ignorar mis propios mensajes
    }
    
    // Procesar solo mensajes de otros
};
```

### **Solución 2: Source 'websocket'**
```javascript
// Al recibir, actualizar con source especial
hot.setDataAtCell(row, col, valor, 'websocket');

// En afterChange, ignorar cambios de WebSocket
afterChange: function(changes, source) {
    if (source !== 'websocket' && source !== 'loadData') {
        // Solo aquí enviar por WebSocket
    }
}
```

---

## 📋 **Estructura de Mensajes SICAM (Propuestas Básicas)**

### **Actualización de Cita**
```javascript
{
    tipo: 'cita_actualizada',
    id_cit: 123,
    campo: 'nom_cit',
    valor: 'Juan Pérez García',
    id_ejecutivo: 5
}
```

### **Nueva Cita**
```javascript
{
    tipo: 'cita_creada',
    id_cit: 456,
    nom_cit: 'María López',
    tel_cit: '555-1234',
    id_ejecutivo: 5
}
```

### **Cita Eliminada**
```javascript
{
    tipo: 'cita_eliminada',
    id_cit: 789,
    nom_cit: 'Cliente Eliminado',
    id_ejecutivo: 5
}
```

> **Nota:** Estas son estructuras **propuestas básicas**. Puedes adaptarlas según las necesidades específicas del módulo.

---

## 🎯 **Integración con Código Existente**

### **En tu `afterChange` actual:**
```javascript
// ANTES
if (changes && source !== 'loadData') {
    // Tu lógica...
}

// DESPUÉS
if (changes && source !== 'loadData' && source !== 'websocket') {
    // Tu lógica...
    
    // + Agregar WebSocket
    const id_cit = hot.getDataAtCell(row, obtenerIndiceColumna('id_cit'));
    const campo = obtenerCampo(column);
    
    socket.send(JSON.stringify({
        tipo: 'cita_actualizada',
        id_cit: id_cit,
        campo: campo,
        valor: newValue,
        id_ejecutivo: idEjecutivoActual
    }));
}
```

---

## ✅ **Ventajas del WebSocket**

1. **Tiempo Real**: Cambios instantáneos sin refrescar
2. **Colaborativo**: Múltiples usuarios trabajando juntos
3. **Eficiente**: No polling continuo al servidor
4. **Visual**: Resaltado automático de cambios externos
5. **Escalable**: Fácil extender a otros módulos

---

## 🚀 **Próximos Pasos**

1. **Probar** los ejemplos básicos
2. **Integrar** con tu Handsontable existente
3. **Extender** a creación y eliminación de citas
4. **Agregar** filtros por plantel/ejecutivo
5. **Implementar** en otros módulos (ejecutivos, planteles)

**El servidor WebSocket ya está listo. Solo necesitas conectarte desde JavaScript.** 🎯

---
# SICAM - Manejo de Archivos en General
## Estándar para subida de imágenes

---

## 📁 ESTRUCTURA DE CARPETAS
```
/SICAM/                         # Nodo raíz del ERP
├── /ejecutivo/                 # Módulo de ejecutivos
│   ├── ejecutivos.php          # Frontend
│   ├── /server/controlador_ejecutivos.php  # Controlador
│   └── /inc/conexion.php       # Configuración BD
└── /uploads/                   # Carpeta de archivos subidos
```

---

## 📋 ESTRUCTURA DE TABLA
```sql
CREATE TABLE ejecutivo (
    id_eje INT PRIMARY KEY AUTO_INCREMENT,
    nom_eje VARCHAR(100) NOT NULL,
    tel_eje VARCHAR(20),
    fot_eje VARCHAR(255)
);
```

---

## 🖥️ HTML - Formulario con Imagen

```html
<!-- Modal para crear ejecutivo -->
<div class="modal fade" id="modal_ejecutivo">
    <div class="modal-dialog">
        <div class="modal-content">
            <div class="modal-header">
                <h4>Crear Ejecutivo</h4>
            </div>
            <div class="modal-body">
                <form id="formularioEjecutivo" enctype="multipart/form-data">
                    
                    <div class="form-group">
                        <label>Nombre:</label>
                        <input type="text" id="nom_eje" name="nom_eje" class="form-control" required>
                    </div>
                    
                    <div class="form-group">
                        <label>Teléfono:</label>
                        <input type="text" id="tel_eje" name="tel_eje" class="form-control" required>
                    </div>
                    
                    <div class="form-group">
                        <label>Foto (opcional):</label>
                        <input type="file" id="fot_eje" name="fot_eje" class="form-control" accept="image/*">
                        <small class="text-muted">JPG, PNG. Máximo 5MB</small>
                    </div>
                    
                    <!-- Preview de imagen -->
                    <div id="preview" style="display:none; margin-top:10px;">
                        <img id="img-preview" src="" style="max-width: 150px; border: 1px solid #ddd;">
                    </div>
                    
                    <div class="form-group" style="margin-top:20px;">
                        <button type="submit" class="btn btn-primary">Guardar</button>
                        <button type="button" class="btn btn-default" data-dismiss="modal">Cancelar</button>
                    </div>
                    
                </form>
            </div>
        </div>
    </div>
</div>
```

---

## 📱 JAVASCRIPT (jQuery) - Validación Frontend

```javascript
// Preview al seleccionar imagen
$("#fot_eje").change(function() {
    mostrarPreview(this);
});

// Submit del formulario con validaciones
$("#formularioEjecutivo").submit(function(e) {
    e.preventDefault();
    
    // Validar campos de texto
    if (!$("#nom_eje").val().trim()) {
        alert('El nombre es requerido');
        return;
    }
    
    if (!$("#tel_eje").val().trim()) {
        alert('El teléfono es requerido');
        return;
    }
    
    // Validar imagen si existe
    if ($("#fot_eje")[0].files[0]) {
        if (!validarImagen()) {
            return;
        }
    }
    
    // Si todo está bien, enviar
    enviarFormulario();
});

// Función para validar imagen
function validarImagen() {
    var archivo = $("#fot_eje")[0].files[0];
    var nombre = archivo.name;
    var tamannio = archivo.size;
    var extension = nombre.split('.').pop().toLowerCase();
    
    // Validar extensión
    if (!['jpg', 'jpeg', 'png'].includes(extension)) {
        alert('Solo se permiten archivos JPG y PNG');
        return false;
    }
    
    // Validar tamaño (5MB)
    if (tamannio > 5242880) {
        alert('La imagen no debe exceder 5MB');
        return false;
    }
    
    return true;
}

// Enviar formulario completo
function enviarFormulario() {
    var formData = new FormData($('#formularioEjecutivo')[0]);
    formData.append('action', 'crear_ejecutivo');
    
    $.ajax({
        url: 'server/controlador_ejecutivos.php',
        type: 'POST',
        data: formData,
        processData: false,
        contentType: false,
        dataType: 'json',
        beforeSend: function() {
            $('button[type="submit"]').prop('disabled', true).text('Guardando...');
        },
        success: function(response) {
            if (response.success) {
                $('#modal_ejecutivo').modal('hide');
                alert(response.message);
                limpiarFormulario();
                // Recargar datos si es necesario
            } else {
                alert('Error: ' + response.message);
            }
        },
        error: function() {
            alert('Error de conexión');
        },
        complete: function() {
            $('button[type="submit"]').prop('disabled', false).text('Guardar');
        }
    });
}

// Mostrar preview de imagen
function mostrarPreview(input) {
    if (input.files && input.files[0]) {
        var reader = new FileReader();
        reader.onload = function(e) {
            $('#img-preview').attr('src', e.target.result);
            $('#preview').show();
        }
        reader.readAsDataURL(input.files[0]);
    } else {
        $('#preview').hide();
    }
}

// Limpiar formulario
function limpiarFormulario() {
    $('#formularioEjecutivo')[0].reset();
    $('#preview').hide();
}
```

---

## ⚙️ PHP - Backend en controlador_ejecutivos.php

```php
case 'crear_ejecutivo':
    $nom_eje = escape($_POST['nom_eje'], $connection);
    $tel_eje = escape($_POST['tel_eje'], $connection);
    
    // Validaciones backend
    if (empty($nom_eje)) {
        echo respuestaError('El nombre es requerido');
        break;
    }
    
    if (empty($tel_eje)) {
        echo respuestaError('El teléfono es requerido');
        break;
    }
    
    // Insertar ejecutivo PRIMERO para obtener el ID
    $query = "INSERT INTO ejecutivo (nom_eje, tel_eje) VALUES ('$nom_eje', '$tel_eje')";
    
    if (!mysqli_query($connection, $query)) {
        echo respuestaError('Error al crear ejecutivo');
        break;
    }
    
    $nuevo_id = mysqli_insert_id($connection);
    
    // Procesar imagen si existe
    if (isset($_FILES['fot_eje']) && $_FILES['fot_eje']['error'] === UPLOAD_ERR_OK) {
        $archivo = $_FILES['fot_eje'];
        $extension = strtolower(pathinfo($archivo['name'], PATHINFO_EXTENSION));
        
        // Validaciones de imagen
        if (!in_array($extension, ['jpg', 'jpeg', 'png'])) {
            echo respuestaError('Solo se permiten archivos JPG y PNG');
            break;
        }
        
        if ($archivo['size'] > 5242880) {
            echo respuestaError('La imagen no debe exceder 5MB');
            break;
        }
        
        // Generar nombre único: ID + SHA-1 del contenido
        $contenido_archivo = file_get_contents($archivo['tmp_name']);
        $sha1_hash = sha1($contenido_archivo . $nuevo_id);
        $fot_eje = "foto-ejecutivo-{$nuevo_id}-{$sha1_hash}.{$extension}";
        
        $ruta = '../../uploads/' . $fot_eje;
        
        // Mover archivo a carpeta uploads
        if (move_uploaded_file($archivo['tmp_name'], $ruta)) {
            // Actualizar ejecutivo con el nombre de la imagen
            $query_update = "UPDATE ejecutivo SET fot_eje = '$fot_eje' WHERE id_eje = '$nuevo_id'";
            mysqli_query($connection, $query_update);
        } else {
            echo respuestaError('Error al guardar imagen');
            break;
        }
    }
    
    echo respuestaExito(['id' => $nuevo_id], 'Ejecutivo creado correctamente');
break;

case 'actualizar_ejecutivo':
    $id_eje = escape($_POST['id_eje'], $connection);
    $nom_eje = escape($_POST['nom_eje'], $connection);
    $tel_eje = escape($_POST['tel_eje'], $connection);
    
    // Validaciones backend
    if (empty($id_eje)) {
        echo respuestaError('ID del ejecutivo es requerido');
        break;
    }
    
    if (empty($nom_eje)) {
        echo respuestaError('El nombre es requerido');
        break;
    }
    
    if (empty($tel_eje)) {
        echo respuestaError('El teléfono es requerido');
        break;
    }
    
    // Actualizar datos básicos
    $query = "UPDATE ejecutivo SET nom_eje = '$nom_eje', tel_eje = '$tel_eje' WHERE id_eje = '$id_eje'";
    
    if (!mysqli_query($connection, $query)) {
        echo respuestaError('Error al actualizar ejecutivo');
        break;
    }
    
    // Procesar nueva imagen si existe
    if (isset($_FILES['fot_eje']) && $_FILES['fot_eje']['error'] === UPLOAD_ERR_OK) {
        $archivo = $_FILES['fot_eje'];
        $extension = strtolower(pathinfo($archivo['name'], PATHINFO_EXTENSION));
        
        // Validaciones de imagen
        if (!in_array($extension, ['jpg', 'jpeg', 'png'])) {
            echo respuestaError('Solo se permiten archivos JPG y PNG');
            break;
        }
        
        if ($archivo['size'] > 5242880) {
            echo respuestaError('La imagen no debe exceder 5MB');
            break;
        }
        
        // Obtener foto actual para eliminarla
        $query_foto = "SELECT fot_eje FROM ejecutivo WHERE id_eje = '$id_eje'";
        $resultado = ejecutarConsulta($query_foto, $connection);
        
        if ($resultado && count($resultado) > 0) {
            $fotoActual = $resultado[0]['fot_eje'];
            
            // Eliminar foto anterior si existe
            if ($fotoActual != NULL && file_exists("../../uploads/$fotoActual")) {
                unlink("../../uploads/$fotoActual");
            }
            
            // Generar NUEVO nombre único (IMPORTANTE: evita caché del navegador)
            $contenido_archivo = file_get_contents($archivo['tmp_name']);
            $sha1_hash = sha1($contenido_archivo . $id_eje . time()); // Agregamos time() para forzar nuevo hash
            $nueva_foto = "foto-ejecutivo-{$id_eje}-{$sha1_hash}.{$extension}";
            
            $ruta = '../../uploads/' . $nueva_foto;
            
            // Mover nueva imagen
            if (move_uploaded_file($archivo['tmp_name'], $ruta)) {
                // Actualizar BD con nuevo nombre
                $query_update = "UPDATE ejecutivo SET fot_eje = '$nueva_foto' WHERE id_eje = '$id_eje'";
                mysqli_query($connection, $query_update);
            } else {
                echo respuestaError('Error al guardar nueva imagen');
                break;
            }
        }
    }
    
    echo respuestaExito(null, 'Ejecutivo actualizado correctamente');
break;
```

---

## 🔧 CARACTERÍSTICAS PRINCIPALES

✅ **Formulario completo**: Datos + imagen en un solo envío  
✅ **Crear y Editar**: Maneja tanto creación como actualización  
✅ **Validación frontend**: jQuery valida antes de enviar  
✅ **Validación backend**: PHP valida nuevamente por seguridad  
✅ **Preview de imagen**: Vista previa antes de enviar  
✅ **Nomenclatura única**: `foto-ejecutivo-{id}-{sha1}.{ext}`  
✅ **Anti-caché**: Rename forzado en ediciones para evitar caché del navegador  
✅ **Limpieza automática**: Elimina imágenes anteriores al actualizar  
✅ **Carpeta centralizada**: Todas las imágenes en `/uploads/`  
✅ **Respuestas JSON**: Formato estándar SICAM  
✅ **Manejo de errores**: Validaciones completas  
✅ **Seguridad**: Escape de datos y validación de archivos

---

## 📝 EJEMPLO DE NOMBRE GENERADO

```
foto-ejecutivo-123-a1b2c3d4e5f6789012345678901234567890abcd.jpg
```

- **123**: ID del ejecutivo en la base de datos
- **a1b2c3d4...**: Hash SHA-1 del contenido del archivo + ID + timestamp
- **jpg**: Extensión original del archivo

**🔒 ¿Cómo evitamos duplicados?**
- El **ID del ejecutivo** es único en la base de datos
- El **SHA-1** se genera del contenido real del archivo + ID + time()
- Si el mismo ejecutivo sube la misma imagen → mismo hash (reemplaza)  
- Si es diferente ejecutivo o diferente imagen → hash único garantizado
- **Imposible colisión**: La combinación ID + SHA-1 + timestamp es matemáticamente única

**⚠️ PROBLEMA DE CACHÉ DEL NAVEGADOR**
La caché es **amiga del usuario** (páginas más rápidas) pero **enemiga del programador** (cambios no se ven).

**🔄 Solución - Rename forzado en edición:**
- Al editar, agregamos `time()` al hash → nuevo nombre siempre  
- Ejemplo: `foto-ejecutivo-123-HASH1.jpg` → `foto-ejecutivo-123-HASH2.jpg`
- El navegador ve nombre diferente → descarga imagen nueva
- **Sin rename = imagen cached antigua, Con rename = imagen nueva**

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
