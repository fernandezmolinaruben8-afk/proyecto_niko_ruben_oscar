# proyecto_bd
Rubén Fernández Molina

Nicolas Ruiz Mengual

Oscar Castaño Cárdenas

Vamos a realizar un proyecto  sobre una funeraria con los siguientes apartados:

1-.Definicion del negocio

2-.Modelo E-R

3-.Modelo Relacional

4-.Script con definicion de BBDD

5-.Vistas

6-.Procedimientos

7-.Disparadores

8-.Ejercicios de Transacciones

[README (1).md](https://github.com/user-attachments/files/27988785/README.1.md)

# PROYECTO BBDD — Funeraria Trujillo

> Proyecto realizado por **Nicolas Ruiz Mengual**, **Oscar Castaño Cárdenas** y **Rubén Fernández Molina**.

---

## Descripción

Funeraria Trujillo es una empresa de servicios exequiales cuya gestión integral se centra en la formalización de contratos personalizados para la atención de fallecidos y el soporte a sus familias. El modelo de negocio se estructura en cuatro pilares fundamentales:

1. Gestión de Clientes y Contratos
2. Servicios y Personal
3. Control de Ubicaciones
4. Trazabilidad de Información

---

## Índice

1. [Modelo E.R.](#1-modelo-er)
2. [Modelo Relacional](#2-modelo-relacional)
3. [Script de Definición BBDD](#3-script-de-definición-bbdd)
4. [Vistas](#4-vistas)
5. [Procedimientos](#5-procedimientos)
6. [Disparadores](#6-disparadores)
7. [Transacciones](#7-transacciones)

---

## 1. Modelo E.R.

### Argumento

- La empresa organiza funerales, por lo que necesita registrar **nombre, apellidos, DNI, teléfono, dirección y email** del cliente que contrata sus servicios.
- Un cliente puede hacer **varios contratos** de defunción.
- De cada contrato se almacena su **estado, total y fecha**.
- Un contrato incluye **un solo difunto**, del que se guarda nombre, apellidos, DNI, fecha de nacimiento, fecha de defunción y causa de muerte.
- Cada difunto tiene **una sola ubicación**, de la que se conoce el sector, la fila, el número y la disponibilidad.
- Un contrato puede contener **muchos servicios**, de los que se registra nombre, descripción y precio base.
- **Muchos empleados** proveen **muchos servicios**; del personal se almacena nombre, apellidos, DNI y cargo.

---

## 2. Modelo Relacional

```
Cliente:    (id_cliente, nombre, apellido1, apellido2, dni, teléfono, dirección, email)
Difunto:    (id_difunto, nombre, apellido1, apellido2, dni, fecha_nacimiento, fecha_defuncion, causa_muerte, id_ubicacion FK)
Ubicación:  (id_ubicacion, sector, fila, número, disponibilidad)
Servicio:   (id_servicio, nombre, descripcion, precio_base)
Personal:   (id_empleado, nombre, apellido1, apellido2, dni, cargo)
Contrato:   (id_contrato, id_cliente, id_difunto, estado, total, fecha_contrato)
CONTIENE:   (id_contrato, id_servicio)
PROVEE:     (id_servicio, id_empleado)
```

### Formas Normales

**1FN** — En las tablas `DIFUNTO`, `CLIENTE` y `PERSONAL`, el campo `apellidos` se divide en `apellido1` y `apellido2` para facilitar búsquedas.

**2FN** — Las relaciones N:M entre `CONTRATO`↔`SERVICIO` y `SERVICIO`↔`PERSONAL` se resuelven con las tablas intermedias `CONTIENE` y `PROVEE` respectivamente.

**3FN** — El campo `total` es un valor calculado (suma de precios de servicios). Técnicamente, para una 3FN estricta, no debería almacenarse directamente en la tabla, ya que depende de los servicios vinculados.

---

## 3. Script de Definición BBDD

```sql
CREATE SCHEMA IF NOT EXISTS funeraria;
USE funeraria;

CREATE TABLE IF NOT EXISTS cliente (
    id_cliente  INT AUTO_INCREMENT PRIMARY KEY,
    nombre      VARCHAR(50)  NOT NULL,
    apellido1   VARCHAR(35)  NOT NULL,
    apellido2   VARCHAR(35),
    dni         VARCHAR(9)   UNIQUE NOT NULL,
    telefono    VARCHAR(20),
    direccion   VARCHAR(100),
    email       VARCHAR(80)
);

CREATE TABLE IF NOT EXISTS ubicacion (
    id_ubicacion  INT AUTO_INCREMENT PRIMARY KEY,
    sector        INT,
    fila          INT,
    numero        INT,
    disponibilidad ENUM('Disponible', 'No disponible') DEFAULT 'Disponible'
);

CREATE TABLE IF NOT EXISTS difunto (
    id_difunto       INT AUTO_INCREMENT PRIMARY KEY,
    nombre           VARCHAR(50)  NOT NULL,
    apellido1        VARCHAR(35)  NOT NULL,
    apellido2        VARCHAR(35),
    dni              VARCHAR(9)   UNIQUE NOT NULL,
    fecha_nacimiento DATE,
    fecha_defuncion  DATE,
    causa_muerte     VARCHAR(255),
    id_ubicacion     INT UNIQUE,
    CONSTRAINT fk_difunto_ubicacion FOREIGN KEY (id_ubicacion)
        REFERENCES ubicacion(id_ubicacion)
);

CREATE TABLE IF NOT EXISTS contrato (
    id_contrato    INT AUTO_INCREMENT PRIMARY KEY,
    id_cliente     INT NOT NULL,
    id_difunto     INT UNIQUE NOT NULL,
    estado         VARCHAR(50),
    fecha_contrato DATE,
    CONSTRAINT fk_contrato_cliente FOREIGN KEY (id_cliente)
        REFERENCES cliente(id_cliente),
    CONSTRAINT fk_contrato_difunto FOREIGN KEY (id_difunto)
        REFERENCES difunto(id_difunto)
);

CREATE TABLE IF NOT EXISTS personal (
    id_empleado INT AUTO_INCREMENT PRIMARY KEY,
    nombre      VARCHAR(50) NOT NULL,
    apellido1   VARCHAR(35) NOT NULL,
    apellido2   VARCHAR(35),
    dni         VARCHAR(9)  UNIQUE NOT NULL,
    cargo       VARCHAR(50)
);

CREATE TABLE IF NOT EXISTS servicio (
    id_servicio  INT AUTO_INCREMENT PRIMARY KEY,
    nombre       VARCHAR(100) NOT NULL,
    descripcion  VARCHAR(125),
    precio_base  DECIMAL(10,2) NOT NULL
);

CREATE TABLE IF NOT EXISTS contiene (
    id_contrato              INT NOT NULL,
    id_servicio              INT NOT NULL,
    cantidad                 INT DEFAULT 1,
    precio_unitario_aplicado DECIMAL(10,2),
    PRIMARY KEY (id_contrato, id_servicio),
    CONSTRAINT fk_dc_contrato FOREIGN KEY (id_contrato) REFERENCES contrato(id_contrato),
    CONSTRAINT fk_dc_servicio FOREIGN KEY (id_servicio) REFERENCES servicio(id_servicio)
);

CREATE TABLE IF NOT EXISTS provee (
    id_servicio INT NOT NULL,
    id_empleado INT NOT NULL,
    PRIMARY KEY (id_servicio, id_empleado),
    CONSTRAINT fk_sp_servicio FOREIGN KEY (id_servicio) REFERENCES servicio(id_servicio),
    CONSTRAINT fk_sp_personal FOREIGN KEY (id_empleado) REFERENCES personal(id_empleado)
);
```

---

## 4. Vistas

### 1. Difuntos con su ubicación

```sql
CREATE OR REPLACE VIEW vista_ubicacion_difuntos AS
SELECT
    d.nombre, d.apellido1, d.dni,
    u.sector, u.fila, u.numero
FROM difunto AS d
LEFT JOIN ubicacion AS u ON d.id_ubicacion = u.id_ubicacion;
```

### 2. Facturación por contrato

```sql
CREATE OR REPLACE VIEW vista_total_contratos AS
SELECT
    co.id_contrato,
    cl.nombre AS cliente,
    SUM(cn.cantidad * cn.precio_unitario_aplicado) AS total_facturado
FROM contrato AS co
JOIN cliente  AS cl ON co.id_cliente  = cl.id_cliente
JOIN contiene AS cn ON co.id_contrato = cn.id_contrato
GROUP BY co.id_contrato;
```

### 3. Personal y servicios

```sql
CREATE OR REPLACE VIEW vista_empleados_servicios AS
SELECT
    p.nombre AS empleado, p.cargo,
    s.nombre AS servicio
FROM personal p
JOIN provee   AS pr ON p.id_empleado  = pr.id_empleado
JOIN servicio AS s  ON pr.id_servicio = s.id_servicio;
```

### 4. Servicios más vendidos

Solo muestra servicios contratados más de 5 veces.

```sql
CREATE OR REPLACE VIEW vista_servicios_populares AS
SELECT
    s.nombre,
    COUNT(cn.id_contrato) AS veces_contratado,
    SUM(cn.cantidad)      AS cantidad_total_vendida
FROM servicio s
JOIN contiene cn ON s.id_servicio = cn.id_servicio
GROUP BY s.id_servicio
HAVING veces_contratado > 5;
```

### 5. Carga de trabajo por empleado

```sql
CREATE OR REPLACE VIEW vista_carga_trabajo_personal AS
SELECT
    p.nombre, p.apellido1, p.cargo,
    COUNT(pr.id_servicio) AS total_servicios_especialidad
FROM personal AS p
LEFT JOIN provee pr ON p.id_empleado = pr.id_empleado
GROUP BY p.id_empleado;
```

---

## 5. Procedimientos

### 1. Registrar un nuevo difunto y asignar ubicación

Inserta un difunto y marca su ubicación como "No disponible".

```sql
DELIMITER //
CREATE PROCEDURE sp_registrar_difunto(
    IN p_nombre       VARCHAR(50),
    IN p_apellido1    VARCHAR(35),
    IN p_apellido2    VARCHAR(35),
    IN p_dni          VARCHAR(9),
    IN p_fecha_nac    DATE,
    IN p_fecha_def    DATE,
    IN p_causa        VARCHAR(255),
    IN p_id_ubicacion INT
)
BEGIN
    INSERT INTO difunto (nombre, apellido1, apellido2, dni,
                         fecha_nacimiento, fecha_defuncion, causa_muerte, id_ubicacion)
    VALUES (p_nombre, p_apellido1, p_apellido2, p_dni,
            p_fecha_nac, p_fecha_def, p_causa, p_id_ubicacion);

    UPDATE ubicacion
    SET disponibilidad = 'No disponible'
    WHERE id_ubicacion = p_id_ubicacion;
END //
DELIMITER ;
```

### 2. Añadir servicio a un contrato

Busca automáticamente el `precio_base` del servicio y lo guarda como `precio_unitario_aplicado`.

```sql
DELIMITER //
CREATE PROCEDURE sp_agregar_servicio_contrato(
    IN p_id_contrato INT,
    IN p_id_servicio INT,
    IN p_cantidad    INT
)
BEGIN
    DECLARE v_precio_actual DECIMAL(10,2);

    SELECT precio_base INTO v_precio_actual
    FROM servicio WHERE id_servicio = p_id_servicio;

    INSERT INTO contiene (id_contrato, id_servicio, cantidad, precio_unitario_aplicado)
    VALUES (p_id_contrato, p_id_servicio, p_cantidad, v_precio_actual);
END //
DELIMITER ;
```

### 3. Eliminar un contrato y sus dependencias

Elimina primero los registros hijos en `contiene` y luego el contrato.

```sql
DELIMITER //
CREATE PROCEDURE sp_eliminar_contrato(IN p_id_contrato INT)
BEGIN
    DELETE FROM contiene WHERE id_contrato = p_id_contrato;
    DELETE FROM contrato WHERE id_contrato = p_id_contrato;
END //
DELIMITER ;
```

### 4. Consultar servicios por empleado

```sql
DELIMITER //
CREATE PROCEDURE sp_servicios_por_empleado(IN p_id_empleado INT)
BEGIN
    SELECT s.nombre, s.descripcion
    FROM servicio s
    JOIN provee pr ON s.id_servicio = pr.id_servicio
    WHERE pr.id_empleado = p_id_empleado;
END //
DELIMITER ;
```

### 5. Actualizar precio de un servicio

```sql
DELIMITER //
CREATE PROCEDURE sp_actualizar_precio_servicio(
    IN p_id_servicio  INT,
    IN p_nuevo_precio DECIMAL(10,2)
)
BEGIN
    UPDATE servicio
    SET precio_base = p_nuevo_precio
    WHERE id_servicio = p_id_servicio;
END //
DELIMITER ;
```

### 6. Generar resumen de contrato (factura simple)

Devuelve datos del cliente, el difunto y el total a pagar.

```sql
DELIMITER //
CREATE PROCEDURE sp_resumen_facturacion_contrato(IN p_id_contrato INT)
BEGIN
    SELECT
        co.id_contrato,
        cl.nombre    AS cliente_nombre,
        cl.apellido1 AS cliente_apellido,
        d.nombre     AS difunto_nombre,
        SUM(cn.cantidad * cn.precio_unitario_aplicado) AS total_a_pagar
    FROM contrato co
    JOIN cliente  cl ON co.id_cliente  = cl.id_cliente
    JOIN difunto  d  ON co.id_difunto  = d.id_difunto
    JOIN contiene cn ON co.id_contrato = cn.id_contrato
    WHERE co.id_contrato = p_id_contrato
    GROUP BY co.id_contrato;
END //
DELIMITER ;
```

---

## 6. Disparadores

### 1. Liberar ubicación al eliminar un difunto

```sql
DELIMITER //
CREATE TRIGGER tr_liberar_ubicacion_delete
AFTER DELETE ON difunto FOR EACH ROW
BEGIN
    UPDATE ubicacion SET disponibilidad = 'Disponible'
    WHERE id_ubicacion = OLD.id_ubicacion;
END //
DELIMITER ;
```

### 2. Validar que no se asigne una ubicación ocupada

```sql
DELIMITER //
CREATE TRIGGER tr_validar_ubicacion_disponible
BEFORE INSERT ON difunto FOR EACH ROW
BEGIN
    DECLARE estado_ubi VARCHAR(20);
    SELECT disponibilidad INTO estado_ubi FROM ubicacion
    WHERE id_ubicacion = NEW.id_ubicacion;
    IF estado_ubi = 'No disponible' THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Error: La ubicación ya está ocupada.';
    END IF;
END //
DELIMITER ;
```

### 3. Evitar precios negativos en servicios

```sql
DELIMITER //
CREATE TRIGGER tr_validar_precio_positivo
BEFORE UPDATE ON servicio FOR EACH ROW
BEGIN
    IF NEW.precio_base < 0 THEN
        SET NEW.precio_base = OLD.precio_base;
    END IF;
END //
DELIMITER ;
```

### 4. Formatear DNI a mayúsculas automáticamente

```sql
DELIMITER //
CREATE TRIGGER tr_cliente_dni_upper
BEFORE INSERT ON cliente FOR EACH ROW
BEGIN
    SET NEW.dni = UPPER(NEW.dni);
END //
DELIMITER ;
```

### 5. Impedir la eliminación de clientes con contratos activos

```sql
DELIMITER //
CREATE TRIGGER tr_prevenir_borrado_cliente
BEFORE DELETE ON cliente FOR EACH ROW
BEGIN
    DECLARE total_contratos INT;
    SELECT COUNT(*) INTO total_contratos FROM contrato
    WHERE id_cliente = OLD.id_cliente;
    IF total_contratos > 0 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'No se puede eliminar un cliente con contratos vinculados.';
    END IF;
END //
DELIMITER ;
```

### 6. Asignación automática de fecha de contrato

```sql
DELIMITER //
CREATE TRIGGER tr_fecha_contrato_default
BEFORE INSERT ON contrato FOR EACH ROW
BEGIN
    IF NEW.fecha_contrato IS NULL THEN
        SET NEW.fecha_contrato = CURDATE();
    END IF;
END //
DELIMITER ;
```

### 7. Control de cantidad mínima en servicios del contrato

```sql
DELIMITER //
CREATE TRIGGER tr_validar_cantidad_servicio
BEFORE INSERT ON contiene FOR EACH ROW
BEGIN
    IF NEW.cantidad <= 0 THEN
        SET NEW.cantidad = 1;
    END IF;
END //
DELIMITER ;
```

### 8. Log de cambios de email de clientes

```sql
DELIMITER //
CREATE TRIGGER tr_log_email_cliente
BEFORE UPDATE ON cliente FOR EACH ROW
BEGIN
    IF OLD.email <> NEW.email THEN
        INSERT INTO log_movimientos (tabla_afectada, accion, fecha, usuario)
        VALUES ('cliente', CONCAT('Email cambiado de ', OLD.email, ' a ', NEW.email), NOW(), USER());
    END IF;
END //
DELIMITER ;
```

### 9. Prevención de ubicaciones sin sector o número

```sql
DELIMITER //
CREATE TRIGGER tr_validar_datos_ubicacion
BEFORE INSERT ON ubicacion FOR EACH ROW
BEGIN
    IF NEW.sector IS NULL OR NEW.numero IS NULL THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Error: El sector y el número son obligatorios para crear una ubicación.';
    END IF;
END //
DELIMITER ;
```

### 10. Sincronizar disponibilidad al cambiar ubicación de difunto

```sql
DELIMITER //
CREATE TRIGGER tr_cambio_ubicacion_difunto
AFTER UPDATE ON difunto FOR EACH ROW
BEGIN
    IF OLD.id_ubicacion <> NEW.id_ubicacion THEN
        UPDATE ubicacion SET disponibilidad = 'Disponible'
        WHERE id_ubicacion = OLD.id_ubicacion;
        UPDATE ubicacion SET disponibilidad = 'No disponible'
        WHERE id_ubicacion = NEW.id_ubicacion;
    END IF;
END //
DELIMITER ;
```

---

## 7. Transacciones

### 1. Registrar difunto, asignar ubicación y crear contrato

```sql
START TRANSACTION;

INSERT INTO difunto (nombre, apellido1, apellido2, dni, fecha_nacimiento, fecha_defuncion, causa_muerte, id_ubicacion)
VALUES ('Juan', 'Pérez', 'García', '12345678Z', '1950-05-20', '2026-05-10', 'Causas naturales', 5);

UPDATE ubicacion SET disponibilidad = 'No disponible' WHERE id_ubicacion = 5;

INSERT INTO contrato (id_cliente, id_difunto, estado, fecha_contrato)
VALUES (1, LAST_INSERT_ID(), 'Activo', CURDATE());

COMMIT;
```

### 2. Añadir múltiples servicios a un contrato

```sql
START TRANSACTION;

INSERT INTO contiene (id_contrato, id_servicio, cantidad, precio_unitario_aplicado)
VALUES (1, 10, 1, 1500.00);

INSERT INTO contiene (id_contrato, id_servicio, cantidad, precio_unitario_aplicado)
VALUES (1, 15, 1, 300.00);

INSERT INTO contiene (id_contrato, id_servicio, cantidad, precio_unitario_aplicado)
VALUES (1, 20, 2, 50.00);

COMMIT;
```

### 3. Traslado de difunto a otra ubicación

```sql
START TRANSACTION;

UPDATE ubicacion SET disponibilidad = 'Disponible'    WHERE id_ubicacion = 5;
UPDATE ubicacion SET disponibilidad = 'No disponible' WHERE id_ubicacion = 12;
UPDATE difunto   SET id_ubicacion = 12                WHERE id_difunto   = 1;

COMMIT;
```

### 4. Eliminar empleado y sus vínculos con servicios

```sql
START TRANSACTION;

DELETE FROM provee   WHERE id_empleado = 5;
DELETE FROM personal WHERE id_empleado = 5;

COMMIT;
```

### 5. Cancelar un contrato y liberar la ubicación

```sql
START TRANSACTION;

SET @difunto_id = (SELECT id_difunto FROM contrato WHERE id_contrato = 10);

UPDATE ubicacion
SET disponibilidad = 'Disponible'
WHERE id_ubicacion = (SELECT id_ubicacion FROM difunto WHERE id_difunto = @difunto_id);

DELETE FROM contiene WHERE id_contrato = 10;
DELETE FROM contrato WHERE id_contrato = 10;

COMMIT;
```
