# PARCIAL II: ANÁLISIS Y DISEÑO DE SOFTWARE
**Asignatura:** Bases de Datos / Programación  
**Proyecto:** Sistema de Gestión e Inventario - Tienda de Electrónica  
**Lenguaje y BD:** Python 3.x + SQLite3  

---

## INTRODUCCIÓN Y CONTEXTO DEL PROBLEMA

La tienda de componentes electrónicos presentaba inconsistencias de inventario causadas por fallos en el registro de operaciones: las ventas se registraban en el sistema, pero el stock no se actualizaba en la base de datos de manera atómica.

Este documento presenta la solución integral al problema mediante el desarrollo de una aplicación en consola estructurada en 3 componentes principales:
1. **Análisis y Diseño del Modelo Relacional** (Modelado conceptual y arquitectónico).
2. **Desarrollo del Sistema CRUD y Consultas** (Implementación en Python/SQLite).
3. **Control Transaccional de Inventario** (Garantía ACID mediante `commit` y `rollback`).

---

## PARTE 1: ANÁLISIS Y DISEÑO DEL SISTEMA (20 Puntos)

### 1. Entidades y Atributos

El sistema se compone de dos entidades principales:

#### A. Entidad `productos` (Catálogo e Inventario)
Representa los artículos electrónicos disponibles para la venta.
* **`id`** (`INTEGER`): Clave primaria, autoincrementable. Identificador único del producto.
* **`nombre`** (`TEXT`): Nombre comercial del producto (campo obligatorio).
* **`precio`** (`REAL`): Precio unitario de venta (debe ser $\ge 0$).
* **`stock`** (`INTEGER`): Cantidad física disponible en inventario (debe ser $\ge 0$).

#### B. Entidad `ventas` (Historial Transaccional)
Almacena el registro histórico de las salidas de almacén.
* **`id`** (`INTEGER`): Clave primaria, autoincrementable. Identificador de la venta.
* **`producto_id`** (`INTEGER`): Clave foránea que referencia a `productos(id)`.
* **`cantidad`** (`INTEGER`): Unidades vendidas en la transacción ($> 0$).
* **`fecha`** (`TEXT`): Marca de tiempo automática en formato `YYYY-MM-DD HH:MM:SS`.

---

### 2. Modelo Relacional y Restricciones de Integridad

#### Diagrama de Entidad-Relación (ASCII)

```text
+-----------------------+          +-----------------------+
|       productos       |          |        ventas         |
+-----------------------+          +-----------------------+
| id (PK)               | 1      N | id (PK)               |
| nombre                |----------| producto_id (FK)      |
| precio                |          | cantidad              |
| stock                 |          | fecha                 |
+-----------------------+          +-----------------------+
```

* **Cardinalidad:** Una relación de Uno a Muchos ($1:N$). Un producto puede figurar en múltiples ventas, pero cada registro de venta pertenece exclusivamente a un solo producto.
* **Integridad Referencial:** Se activa explícitamente en SQLite con `PRAGMA foreign_keys = ON;`.
* **Restricción de Eliminación:** Se aplica `ON DELETE RESTRICT` en la clave foránea. Esto impide borrar un producto de la tabla `productos` si ya tiene ventas registradas en la tabla `ventas`, protegiendo la integridad de los reportes históricos.

#### DDL (Data Definition Language) en SQL

```sql
-- Tabla de Productos
CREATE TABLE IF NOT EXISTS productos (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    nombre TEXT NOT NULL,
    precio REAL NOT NULL,
    stock INTEGER NOT NULL
);

-- Tabla de Ventas
CREATE TABLE IF NOT EXISTS ventas (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    producto_id INTEGER NOT NULL,
    cantidad INTEGER NOT NULL,
    fecha TEXT NOT NULL,
    FOREIGN KEY (producto_id) REFERENCES productos(id) ON DELETE RESTRICT
);
```

---

### 3. Diagrama de Arquitectura de la Aplicación

```text
+------------------------+       +------------------------+       +------------------------+
|        USUARIO         |       |    PROGRAMA PYTHON     |       |    BASE DE DATOS BD    |
+------------------------+       +------------------------+       +------------------------+
|                        |       |  - Menú interactivo    |       |  tienda_electronica.db |
|  - Ingresa opciones    | ----> |  - Validaciones (Input)| ----> |                        |
|  - Digita datos        |       |  - sqlite3 Driver      |       |  - Tabla productos     |
|  - Recibe respuestas   | <---- |  - Control Transacción | <---- |  - Tabla ventas        |
|                        |       |    (BEGIN/COMMIT)      |       |                        |
+------------------------+       +------------------------+       +------------------------+
```

#### Flujo de Información:
1. **Entrada:** El usuario selecciona una opción en la consola (p. ej., "Registrar venta").
2. **Procesamiento:** Python captura las entradas, valida rangos numéricos y estructura la consulta.
3. **Persistencia SQL:** Se ejecutan sentencias SQL parametrizadas (`?`) sobre la base de datos `tienda_electronica.db`.
4. **Salida:** Se retorna la confirmación o la tabla formateada al usuario.

---

### 4. Flujo Detallado para Registrar una Venta

Para solucionar la inconsistencia de inventario, el registro de ventas sigue una secuencia estricta:

```text
[Inicio]
   │
   ▼
1. Pedir ID del producto y Cantidad a vender.
   │
   ▼
2. ¿El ID existe en la tabla 'productos'?
   ├── NO ──> Mostrar error y Cancelar.
   └── SÍ
        │
        ▼
3. ¿El Stock actual es >= Cantidad solicitada?
   ├── NO ──> Mostrar error ("Stock insuficiente") y Cancelar.
   └── SÍ
        │
        ▼
4. Iniciar Transacción (BEGIN TRANSACTION).
        │
        ▼
5. Insertar fila en tabla 'ventas'.
        │
        ▼
6. Actualizar (Restar stock) en tabla 'productos'.
        │
        ├── ¿Ocurrió algún error en el proceso?
        │      ├── SÍ ──> Ejecutar ROLLBACK (Cancela todo) ──> Reportar Error.
        │      └── NO
        ▼
7. Ejecutar COMMIT (Guarda todo de forma permanente).
        │
        ▼
[Fin - Venta Existosa]
```

---

## PARTE 2: DESARROLLO DEL PROGRAMA EN PYTHON (50 Puntos)

La aplicación implementa las siguientes funciones organizadas modularmente en el script `VentasSQL.py`:

1. **`inicializar_base_datos()`**: Crea la base de datos `tienda_electronica.db` y sus tablas si no existen.
2. **`registrar_producto()`**: Inserta nuevos productos validando que el precio y el stock no sean negativos.
3. **`consultar_productos()`**: Consulta la tabla `productos` y formatea la salida en una tabla limpia por consola.
4. **`actualizar_producto()`**: Permite modificar el precio o el stock de un producto existente seleccionado por su `id`.
5. **`eliminar_producto()`**: Elimina un producto. Si el producto tiene ventas vinculadas, se activa la restricción `ON DELETE RESTRICT` y Python captura el `sqlite3.IntegrityError` mostrando un aviso.
6. **`mostrar_historial_ventas()`**: Ejecuta una consulta con la cláusula `JOIN` entre `ventas` y `productos` para presentar el nombre del producto en lugar de solo su ID:
   ```sql
   SELECT v.id, p.nombre, v.cantidad, v.fecha 
   FROM ventas v
   JOIN productos p ON v.producto_id = p.id;
   ```
7. **Consultas Parametrizadas:** Toda interacción SQL utiliza el comodín `?` (ejemplo: `WHERE id = ?`) para prevenir vulnerabilidades de Inyección SQL.

---

## PARTE 3: MANEJO DE TRANSACCIONES Y RETO ROLLBACK (30 Puntos)

### Concepto de Atomicidad (ACID)
En sistemas de bases de datos, el principio de **Atomicidad** dictamina que una operación compuesta debe ejecutarse completamente o no ejecutarse en absoluto.

Registrar una venta requiere **dos pasos operacionales obligatorios**:
* **Paso A:** `INSERT INTO ventas ...`
* **Paso B:** `UPDATE productos SET stock = stock - cantidad ...`

Si el Paso A se completa pero el Paso B falla, la base de datos queda corrupta.

### Solución con `commit()` y `rollback()`
* **`conexion.commit()`**: Se ejecuta **únicamente cuando ambas operaciones tienen éxito**. Guarda los cambios definitivamente en el disco.
* **`conexion.rollback()`**: Se ejecuta dentro del bloque `except` si ocurre cualquier fallo intermedio. Revierte los cambios temporales realizados durante la transacción activa, devolviendo la base de datos a su estado exacto inicial.

---

### Demostración del Reto Rollback (`demostrar_reto_rollback`)

El programa incluye la opción **7** en el menú principal para demostrar este control de errores mediante una prueba simulada:

#### Código de la prueba:
```python
# 1. Iniciar transacción explícita
conexion.execute("BEGIN TRANSACTION;")

# 2. Operación A: Insertar la venta (Temporal)
cursor.execute("INSERT INTO ventas (producto_id, cantidad, fecha) VALUES (?, ?, ?)", ...)

# 3. Simulación de fallo de red / corte eléctrico
raise RuntimeError("Fallo simulado del sistema antes de actualizar inventario.")

# 4. Operación B (Inalcanzable por el fallo)
# cursor.execute("UPDATE productos SET stock = ...")

# 5. Captura del fallo y marcha atrás
except Exception as e:
    conexion.rollback() # Revierte el INSERT del paso 2
```

#### Resultado Comprobado de la Prueba:
1. El programa consulta el stock inicial (ejemplo: **10 unidades**).
2. Se inserta la venta temporalmente dentro de la transacción.
3. Se fuerza la excepción `RuntimeError`.
4. Se ejecuta el bloque `except` llamando a `conexion.rollback()`.
5. Se consulta de nuevo el stock final: **permanece intacto en 10 unidades** y el registro de la venta no fue almacenado.