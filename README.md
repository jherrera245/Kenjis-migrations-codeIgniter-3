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
$config['migration_auto_latest'] = FALSE;  // Automáticamente actualizar a la última versión cambiar a TRUE
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
filename="${timestamp}_$1.php"

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

# Agregar contenido básico al archivo de migración
cat <<EOL > "$filepath"
<?php
defined('BASEPATH') OR exit('No direct script access allowed');

class Migration_$1 extends CI_Migration {

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
./create_migration.sh create_users_table
```

Buscar la migración en la caperta migration tendra un nombre similar a este **20240906223232_Create_users.php**, agregaremos un par de columnas
```sh
<?php

class Migration_create_users_table extends CI_Migration {

    public function up()
    {
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
                'null' => TRUE,
            ),
            'updated_at' => array(
                'type' => 'DATETIME',
                'null' => TRUE,
            ),
        ));
        
        // Primary Key
        $this->dbforge->add_key('id', TRUE);
        
        // Crear la tabla
        $this->dbforge->create_table('users');
    }

    public function down()
    {
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
        $this->load->database();
    }

    public function index() {
        $action = $this->input->get('migrate');
        $version = $this->input->get('version');

        switch ($action) {
            case 'latest':
                // Ejecutar todas las migraciones pendientes
                $migrations = $this->migration->find_migrations(); // Obtén las migraciones pendientes
                if ($this->migration->latest() === FALSE) {
                    echo $this->migration->error_string();
                } else {
                    echo 'Migraciones ejecutadas hasta la versión más reciente.';
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

            case 'status':
                // Mostrar el estado de las migraciones ejecutadas
                $migrations = $this->db->get('migrations')->result();
                if (empty($migrations)) {
                    echo 'No se encontraron migraciones ejecutadas.';
                } else {
                    foreach ($migrations as $migration) {
                        echo "Migración ejecutada: Versión " . $migration->version . "\n";
                    }
                }
                break;

            case 'list':
                // Listar los archivos de migración ejecutados
                $this->db->select('migration');
                $this->db->from('migrations');
                $executed_migrations = $this->db->get()->result();
                if (empty($executed_migrations)) {
                    echo 'No se encontraron archivos de migración ejecutados.';
                } else {
                    foreach ($executed_migrations as $migration) {
                        echo "Archivo de migración ejecutado: " . $migration->migration . "\n";
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

8) Creación de archivo bash para ejecutar las migraciones desde la consola **migrate.sh**
```sh
#!/bin/bash

# Verificar si se pasa el comando (migrate, rollback, reset, status)
if [ -z "$1" ]; then
    echo "Error: Debes especificar un comando (migrate, rollback, reset, status)."
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
    status)
        echo "Obteniendo el estado de las migraciones..."
        response=$(curl -s -X GET "${BASE_URL}/${CI_INDEX}?migrate=status")
        echo "$response"
        ;;
    *)
        echo "Comando no reconocido. Usa 'migrate', 'rollback <version>', 'reset' o 'status'."
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

class Migration_add_columns_to_table extends CI_Migration {

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

11) Ejemplo de tablas relacionales
```sh
<?php
defined('BASEPATH') OR exit('No direct script access allowed');

class Migration_create_posts_table extends CI_Migration {

    public function up() {
        // Crear la tabla
        $this->dbforge->add_field(array(
            'id' => array(
                'type' => 'INT',
                'constraint' => 11,
                'unsigned' => TRUE,
                'auto_increment' => TRUE
            ),
            'user_id' => array(
                'type' => 'INT',
                'constraint' => 11,
                'unsigned' => TRUE,
            ),
            'title' => array(
                'type' => 'VARCHAR',
                'constraint' => '255',
            ),
            'body' => array(
                'type' => 'TEXT',
                'null' => TRUE,
            ),
            'created_at' => array(
                'type' => 'DATETIME',
                'null' => TRUE,
            ),
            'updated_at' => array(
                'type' => 'DATETIME',
                'null' => TRUE,
            ),
        ));
        
        // Clave primaria
        $this->dbforge->add_key('id', TRUE);
        
        // Crear la tabla sin la clave foránea
        $this->dbforge->create_table('posts');
        
        // Agregar la clave foránea con una consulta SQL
        $this->db->query('ALTER TABLE posts ADD CONSTRAINT post_fk_user_id  FOREIGN KEY (user_id) REFERENCES users(id)');
    }

    public function down() {
        // Código para revertir los cambios (rollback)
        // Primero eliminamos la clave foránea
        $this->db->query('ALTER TABLE posts DROP FOREIGN KEY post_fk_user_id');
        
        // Luego eliminamos la tabla
        $this->dbforge->drop_table('posts');
    }
}
```

13) Ejemplo usando otros tipos de datos
```sh
<?php

class Migration_create_example_table extends CI_Migration {

    public function up()
    {
        $this->dbforge->add_field(array(
            // Entero
            'id' => array(
                'type' => 'INT',
                'constraint' => 11,
                'unsigned' => TRUE,
                'auto_increment' => TRUE
            ),
            // Cadena de longitud variable
            'name' => array(
                'type' => 'VARCHAR',
                'constraint' => '255',
            ),
            // Texto largo
            'description' => array(
                'type' => 'TEXT',
                'null' => TRUE,
            ),
            // Fecha
            'created_date' => array(
                'type' => 'DATE',
            ),
            // Fecha y hora
            'created_at' => array(
                'type' => 'DATETIME',
                'null' => TRUE,
            ),
            // Booleano
            'is_active' => array(
                'type' => 'BOOLEAN',
                'default' => TRUE,
            ),
            // Decimal
            'price' => array(
                'type' => 'DECIMAL',
                'constraint' => '10,2', // 10 dígitos totales, 2 después del punto decimal
                'default' => '0.00',
            ),
            // Enumeración
            'status' => array(
                'type' => 'ENUM',
                'constraint' => ['pending', 'completed', 'failed'],
                'default' => 'pending',
            ),
            // Binario grande
            'file_data' => array(
                'type' => 'BLOB',
                'null' => TRUE,
            ),
        ));
        
        // Definir clave primaria
        $this->dbforge->add_key('id', TRUE);

        // Crear la tabla
        $this->dbforge->create_table('example_table');
    }

    public function down()
    {
        // Eliminar la tabla si se deshace la migración
        $this->dbforge->drop_table('example_table');
    }
}
```
