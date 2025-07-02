🛠️ Stack Tecnológico

PHP 5.6 (Paradigma funcional/imperativo)
MySQL con mysqli_* exclusivamente
jQuery + Bootstrap
Charset: UTF-8 (utf8mb4)
Sin frameworks


📁 Estructura Modular
/ejecutivos/
├── 📄 citas.php                    # Frontend
├── 📁 server/
│   └── 📄 controlador_citas.php    # Controlador MySQL
└── 📁 inc/
    └── 📄 conexion.php             # Configuración BD

⚠️ Nota: Todos los archivos van en plural (ej: controlador_citas.php)


⚙️ Configuración Base
📋 /inc/conexion.php
php<?php
$host = "localhost";
$user = "sicam_user"; 
$pass = "sicam_pass";
$database = "sicam_db";
$connection = mysqli_connect($host, $user, $pass, $database);

if (!$connection) {
    die('Error de conexión: ' . mysqli_connect_error());
}

mysqli_set_charset($connection, "utf8mb4");

// Función base para ejecutar consultas
function ejecutarConsulta($query, $connection) {
   $result = mysqli_query($connection, $query);
   if (!$result) return false;
   $datos = [];
   while($row = mysqli_fetch_assoc($result)) {
       $datos[] = $row;
   }
   return $datos;
}

// Función para escape rápido
function escape($valor, $connection) {
    return mysqli_real_escape_string($connection, $valor);
}

// Respuestas JSON estándar
function respuestaExito($data = null, $message = 'OK') {
    return json_encode([
        'success' => true,
        'data' => $data,
        'message' => $message
    ], JSON_UNESCAPED_UNICODE);
}

function respuestaError($message = 'Error', $code = 400) {
    return json_encode([
        'success' => false,
        'message' => $message,
        'code' => $code
    ], JSON_UNESCAPED_UNICODE);
}
?>

🔄 Integración AJAX Frontend
📱 Patrón estándar con jQuery
javascript$.ajax({
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

function renderizarCitas(citas) {
    var html = '';
    citas.forEach(function(cita) {
        html += '<div>' + cita.nom_cit + ' - ' + cita.tel_cit + ' (Ejecutivo: ' + cita.nom_eje + ')</div>';
    });
    $('#contenedor-citas').html(html);
}

🔧 Controlador Backend
📡 /server/controlador_citas.php
php<?php
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
    }
    
    mysqli_close($connection);
    exit;
}
?>

🖼️ Frontend Básico
📄 citas.php
html<div id="contenedor-citas"></div>

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

🧱 Template Renderer
🎨 Cuando necesitas PHP + HTML
php<?php
    $datos = json_decode($_POST['datos'], true);
    foreach($datos as $item):
?>
        <div>
            <?= $item['nom_cit'] ?> - <?= $item['tel_cit'] ?>
        </div>
<?php endforeach; ?>

✅ Reglas Importantes

✅ Header Content-Type con charset=utf-8 en controladores
✅ Controlador devuelve JSON con array estructurado
✅ response.data es un array de objetos
✅ escape() obligatorio para prevenir SQL Injection
✅ Funciones pequeñas (máximo 20-30 líneas)
✅ Separar lógica de presentación
✅ Mostrar query en errores para debugging
✅ Vibecodear moderadamente 🚀


🚀 Crear Nuevo Módulo
📋 Pasos rápidos:

Crear estructura:
/mi_modulo/
├── 📄 mi_modulo.php
├── 📁 server/
│   └── 📄 controlador_mi_modulo.php
└── 📁 inc/
    └── 📄 conexion.php

Copiar conexion.php base
Adaptar controlador con tu lógica
Frontend con AJAX estándar
