# EVENTSMSQL
el desarrollo de esta em,pieza en la linea 27
Actividad
Haciendo uso de las siguientes tablas para la base de datos de pizza realice los siguientes ejercicios de Events centrados en el uso de ON COMPLETION PRESERVE y ON COMPLETION NOT PRESERVE :

CREATE TABLE IF NOT EXISTS resumen_ventas (
fecha       DATE      PRIMARY KEY,
total_pedidos INT,
total_ingresos DECIMAL(12,2),
creado_en DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS alerta_stock (
  id              INT AUTO_INCREMENT PRIMARY KEY,
  ingrediente_id  INT UNSIGNED NOT NULL,
  stock_actual    INT NOT NULL,
  fecha_alerta    DATETIME NOT NULL,
  creado_en DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (ingrediente_id) REFERENCES ingrediente(id)
);
Resumen Diario Único : crear un evento que genere un resumen de ventas una sola vez al finalizar el día de ayer y luego se elimine automáticamente llamado ev_resumen_diario_unico.
Resumen Semanal Recurrente: cada lunes a las 01:00 AM, generar el total de pedidos e ingresos de la semana pasada, manteniendo el evento para que siga ejecutándose cada semana llamado ev_resumen_semanal.
Alerta de Stock Bajo Única: en un futuro arranque del sistema (requerimiento del sistema), generar una única pasada de alertas (alerta_stock) de ingredientes con stock < 5, y luego autodestruir el evento.
Monitoreo Continuo de Stock: cada 30 minutos, revisar ingredientes con stock < 10 e insertar alertas en alerta_stock, dejando el evento activo para siempre llamado ev_monitor_stock_bajo.
Limpieza de Resúmenes Antiguos: una sola vez, eliminar de resumen_ventas los registros con fecha anterior a hace 365 días y luego borrar el evento llamado ev_purgar_resumen_antiguo.
------------------------------------------------------------------------------------------------------------------------------------------------------------------
CREATE DATABASE UIO;
USE UIO;
CREATE TABLE IF NOT EXISTS ingrediente (
  id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  nombre VARCHAR(100),
  stock INT,
  unidad VARCHAR(20),
  creado_en DATETIME DEFAULT CURRENT_TIMESTAMP
);
CREATE TABLE IF NOT EXISTS resumen_ventas (
fecha       DATE      PRIMARY KEY,
total_pedidos INT,
total_ingresos DECIMAL(12,2),
creado_en DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS alerta_stock (
  id              INT AUTO_INCREMENT PRIMARY KEY,
  ingrediente_id  INT UNSIGNED NOT NULL,
  stock_actual    INT NOT NULL,
  fecha_alerta    DATETIME NOT NULL,
  creado_en DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (ingrediente_id) REFERENCES ingrediente(id)
);
CREATE TABLE IF NOT EXISTS pedidos (
  id INT AUTO_INCREMENT PRIMARY KEY,
  fecha DATE NOT NULL,
  total DECIMAL(12,2) NOT NULL,
  creado_en DATETIME DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO pedidos (fecha, total) VALUES
(CURDATE() - INTERVAL 1 DAY, 100.50),
(CURDATE() - INTERVAL 1 DAY, 85.00),
(CURDATE() - INTERVAL 3 DAY, 45.75),
(CURDATE() - INTERVAL 8 DAY, 99.90),
(CURDATE() - INTERVAL 7 DAY, 60.00),
(CURDATE(), 120.00);

DROP EVENT IF EXISTS ev_resumen_diario_unico;
DELIMITER //
CREATE EVENT ev_resumen_diario_unico
ON SCHEDULE AT CURRENT_DATE + INTERVAL 10 MINUTE
ON COMPLETION NOT PRESERVE
DO
BEGIN
  INSERT INTO resumen_ventas (fecha, total_pedidos, total_ingresos, creado_en)
  SELECT
    CURDATE() - INTERVAL 1 DAY AS fecha,
    COUNT(*) AS total_pedidos,
    SUM(total) AS total_ingresos,
    NOW();
END //
DELIMITER ;


DROP EVENT IF EXISTS ev_resumen_semanal;

DELIMITER //
CREATE EVENT ev_resumen_semanal
ON SCHEDULE
  EVERY 1 WEEK
  STARTS (
    CURRENT_DATE + INTERVAL ((8 - DAYOFWEEK(CURRENT_DATE)) % 7) DAY + INTERVAL 1 HOUR
  )
ON COMPLETION PRESERVE
DO
BEGIN
  INSERT INTO resumen_ventas (fecha, total_pedidos, total_ingresos, creado_en)
  SELECT
    CURDATE() - INTERVAL ((DAYOFWEEK(CURDATE()) + 5) % 7 + 1) DAY AS fecha,
    COUNT(*) AS total_pedidos,
    SUM(total) AS total_ingresos,
    NOW()
  FROM pedidos
  WHERE fecha >= CURDATE() - INTERVAL ((DAYOFWEEK(CURDATE()) + 5) % 7 + 7) DAY
    AND fecha < CURDATE() - INTERVAL ((DAYOFWEEK(CURDATE()) + 5) % 7) DAY;
END //
DELIMITER ;


DROP EVENT IF EXISTS ev_alerta_stock_bajo_unica;

DELIMITER //

CREATE EVENT ev_alerta_stock_bajo_unica
ON SCHEDULE AT NOW() + INTERVAL 1 MINUTE
ON COMPLETION NOT PRESERVE
DO
BEGIN
  INSERT INTO alerta_stock (ingrediente_id, stock_actual, fecha_alerta, creado_en)
  SELECT id, stock, NOW(), NOW()
  FROM ingrediente
  WHERE stock < 5;
END //

DELIMITER ;


DROP EVENT IF EXISTS ev_monitor_stock_bajo;

DELIMITER //

CREATE EVENT ev_monitor_stock_bajo
ON SCHEDULE EVERY 30 MINUTE
STARTS NOW()
ON COMPLETION PRESERVE
DO
BEGIN
  INSERT INTO alerta_stock (ingrediente_id, stock_actual, fecha_alerta, creado_en)
  SELECT id, stock, NOW(), NOW()
  FROM ingrediente
  WHERE stock < 10;
END //

DELIMITER ;




DROP EVENT IF EXISTS ev_purgar_resumen_antiguo;

DELIMITER //

CREATE EVENT ev_purgar_resumen_antiguo
ON SCHEDULE AT NOW() + INTERVAL 1 MINUTE
ON COMPLETION NOT PRESERVE
DO
BEGIN
  DELETE FROM resumen_ventas
  WHERE fecha < CURDATE() - INTERVAL 365 DAY;
END //

DELIMITER ;
