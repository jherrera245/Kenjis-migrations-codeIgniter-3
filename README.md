# Kenjis migrations codeIgniter 3
Implementación de un gestor de migraciones usando kenjis en codeligniter 3

1) Descargar cli
```sh
composer require kenjis/codeigniter-cli --dev
```

2) Instale el archivo de comando ( cli) y los archivos de configuración ( config/) en su proyecto CodeIgniter:
```sh
php vendor/kenjis/codeigniter-cli/install.php
```

3) Configurar las migraciones en config/migration.php
```sh
<?php
defined('BASEPATH') OR exit('No direct script access allowed');

$config['migration_enabled'] = TRUE;  // Habilitar migraciones
$config['migration_type'] = 'timestamp';  // Puede ser 'sequential' o 'timestamp'
$config['migration_table'] = 'migrations';  // Tabla donde se almacenan las migraciones
$config['migration_auto_latest'] = FALSE;  // Automáticamente actualizar a la última versión
$config['migration_version'] = 0;  // Número de la versión de migración (0 para resetear)
$config['migration_path'] = APPPATH . 'migrations/';  // Ruta de la carpeta de migraciones
```

4) Crear una carpeta de migraciones
```sh
mkdir application/migrations
```

5) Crea un archivo de script Bash llamado **create_migration.sh**.
```sh
touch create_migration.sh
chmod +x create_migration.sh  # Hacer el script ejecutable
```

Contenido
```sh
#!/bin/bash

# Función para mostrar el uso del script
function usage() {
    echo "Uso: $0 <nombre_migracion>"
    echo "Ejemplo: $0 create_users"
    exit 1
}

# Verificar si se pasó el nombre de la migración como argumento
if [ -z "$1" ]; then
    echo "Error: Debes especificar el nombre de la migración."
    usage
fi

# Obtener el timestamp actual
timestamp=$(date +"%Y%m%d%H%M%S")

# Definir el nombre del archivo basado en el timestamp y el nombre dado
migration_name=$1
filename="${timestamp}_${migration_name}.php"

# Ruta de la carpeta de migraciones
migration_path="application/migrations/"

# Verificar si la carpeta de migraciones existe
if [ ! -d "$migration_path" ]; then
    echo "Error: La carpeta de migraciones '$migration_path' no existe."
    exit 1
fi

# Crear el archivo de migración en la carpeta correspondiente
filepath="${migration_path}${filename}"
touch "$filepath"

# Verificar si el archivo se creó correctamente
if [ ! -f "$filepath" ]; then
    echo "Error: No se pudo crear el archivo de migración."
    exit 1
fi

# Convertir el nombre de la migración a PascalCase para la clase PHP
# Reemplazar guiones bajos por espacios, poner en mayúscula la primera letra de cada palabra, y eliminar espacios.
class_name=$(echo "$migration_name" | sed -r 's/(^|_)([a-z])/\U\2/g')

# Agregar contenido básico al archivo de migración
cat <<EOL > "$filepath"
<?php
defined('BASEPATH') OR exit('No direct script access allowed');

class Migration_$class_name extends CI_Migration {

    public function up() {
        // Código para crear la tabla o modificar la base de datos
    }

    public function down() {
        // Código para revertir los cambios (rollback)
    }
}
EOL

# Confirmación
echo "Migración creada: $filepath"
```

6) Ejemplo: Crear una migración para una tabla users
```sh
./create_migration.sh create_users
```

Buscar la migración en la caperta migration tendra un nombre similar a este **20240906223232_Create_users.php**, agregaremos un par de columnas
```sh
<?php
defined('BASEPATH') OR exit('No direct script access allowed');

class Migration_CreateUsers extends CI_Migration {

    public function up() {
        // Definir la estructura de la tabla users
        $this->dbforge->add_field(array(
            'id' => array(
                'type' => 'INT',
                'constraint' => 11,
                'unsigned' => TRUE,
                'auto_increment' => TRUE
            ),
            'username' => array(
                'type' => 'VARCHAR',
                'constraint' => '100',
            ),
            'email' => array(
                'type' => 'VARCHAR',
                'constraint' => '100',
            ),
            'password' => array(
                'type' => 'VARCHAR',
                'constraint' => '255',
            ),
            'created_at' => array(
                'type' => 'DATETIME',
            ),
        ));
        $this->dbforge->add_key('id', TRUE);
        $this->dbforge->create_table('users');
    }

    public function down() {
        // Eliminar la tabla users en caso de hacer rollback
        $this->dbforge->drop_table('users');
    }
}
```

7) Crear controlador para ejecutar las migraciones.
```sh
<?php
defined('BASEPATH') OR exit('No direct script access allowed');

class Migrate extends CI_Controller {

    public function __construct() {
        parent::__construct();
        $this->load->library('migration');
    }

    public function index() {
        $action = $this->input->get('migrate');
        $version = $this->input->get('version');

        switch ($action) {
            case 'latest':
                // Ejecutar todas las migraciones pendientes
                if ($this->migration->latest() === FALSE) {
                    echo $this->migration->error_string();
                } else {
                    echo 'Migraciones ejecutadas correctamente.';
                }
                break;

            case 'version':
                if ($version === NULL) {
                    echo 'Error: Debes especificar el número de versión.';
                } elseif ($version == 0) {
                    // Revertir todas las migraciones
                    if ($this->migration->version(0) === FALSE) {
                        echo $this->migration->error_string();
                    } else {
                        echo 'Todas las migraciones revertidas correctamente.';
                    }
                } else {
                    // Hacer rollback a una versión específica
                    if ($this->migration->version($version) === FALSE) {
                        echo $this->migration->error_string();
                    } else {
                        echo "Rollback a la versión $version ejecutado correctamente.";
                    }
                }
                break;

            default:
                echo 'Comando no reconocido.';
                break;
        }
    }
}
```

8) Creación de archivo bash para ejecutar las migraciones desde la consola migrate.sh
```sh
#!/bin/bash

# Verificar si se pasa el comando (migrate, rollback, reset)
if [ -z "$1" ]; then
    echo "Error: Debes especificar un comando (migrate, rollback, reset)."
    exit 1
fi

# Configurar las variables de entorno
BASE_URL='http://localhost'
CI_INDEX="index.php/migrate"

# Ejecutar el comando correspondiente
case "$1" in
    migrate)
        echo "Ejecutando migraciones..."
        response=$(curl -s -X GET "${BASE_URL}/${CI_INDEX}?migrate=latest")
        echo "$response"
        ;;
    rollback)
        if [ -z "$2" ]; then
            echo "Error: Debes especificar el número de versión para el rollback."
            exit 1
        fi
        echo "Haciendo rollback a la versión $2..."
        response=$(curl -s -X GET "${BASE_URL}/${CI_INDEX}?migrate=version&version=$2")
        echo "$response"
        ;;
    reset)
        echo "Revirtiendo todas las migraciones..."
        response=$(curl -s -X GET "${BASE_URL}/${CI_INDEX}?migrate=version&version=0")
        echo "$response"
        ;;
    *)
        echo "Comando no reconocido. Usa 'migrate', 'rollback <version>', o 'reset'."
        exit 1
        ;;
esac
```

9) Ejecutar las migraciones

Para ejecutar todas las migraciones:
```sh
./migrate.sh migrate
```

Para hacer rollback a una versión específica:
```sh
./migrate.sh rollback 20240905123456
```

Para revertir todas las migraciones:
```sh
./migrate.sh reset
```

10) Agregar columnas a una tabla creamos una nueva migracion
```sh
./create_migration.sh add_columns_to_table
```

Ejemplo
```sh
<?php

class Migration_AddColumnsToTable extends CI_Migration {

    public function up() {
        // Agregar nuevas columnas a la tabla 'your_table'
        $fields = array(
            'new_column1' => array(
                'type' => 'VARCHAR',
                'constraint' => '100',
                'null' => TRUE,
            ),
            'new_column2' => array(
                'type' => 'INT',
                'constraint' => 11,
                'unsigned' => TRUE,
                'null' => TRUE,
            ),
        );

        // Modificar la tabla para agregar las nuevas columnas
        $this->dbforge->add_column('your_table', $fields);
    }

    public function down() {
        // Eliminar las columnas agregadas
        $this->dbforge->drop_column('your_table', 'new_column1');
        $this->dbforge->drop_column('your_table', 'new_column2');
    }
}
```
