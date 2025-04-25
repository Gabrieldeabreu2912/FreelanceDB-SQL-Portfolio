# FreelanceDB-SQL-Portfolio
Comprehensive SQL database schema for a freelance project management platform, built as a portfolio piece. Showcases advanced techniques: normalization, complex constraints, diverse indexing (incl. Full-Text), triggers, stored procedures/functions, JSON types, detailed auditing via history table, and optimistic concurrency control.

-- --------------------------------------------------------------------------
-- Script SQL para Portafolio - Base de Datos FreelanceProManager (v3 - Avanzado)
-- Demuestra: Soluciones a problemas comunes, Auditoría Detallada, Procedimientos/Funciones,
--            Full-Text Search, JSON, Ejemplos de Consultas Avanzadas.
-- Motor de BD Orientativo: MySQL / MariaDB (v5.6+ para JSON, v5.7+/10.0.5+ para Generated Columns)
-- --------------------------------------------------------------------------

-- SECCIÓN 0: Limpieza Inicial
-- --------------------------------------------------------------------------
DROP VIEW IF EXISTS `vista_proyectos_activos_detalle`;
DROP TRIGGER IF EXISTS `trg_proyecto_after_update_audit`;
DROP TRIGGER IF EXISTS `trg_proyecto_after_insert_audit`;
DROP TRIGGER IF EXISTS `trg_proyecto_after_delete_audit`;
DROP TRIGGER IF EXISTS `trg_proyecto_before_update_concurrency`; -- Renombrado el trigger anterior
DROP PROCEDURE IF EXISTS `sp_AsignarFreelancerTarea`;
DROP FUNCTION IF EXISTS `fn_NombreCompletoFreelancer`;
DROP TABLE IF EXISTS `HistorialCambios`;
DROP TABLE IF EXISTS `Pagos`;
DROP TABLE IF EXISTS `AsignacionesTareas`;
DROP TABLE IF EXISTS `Tareas`;
DROP TABLE IF EXISTS `Proyectos`;
DROP TABLE IF EXISTS `CategoriasProyectos`;
DROP TABLE IF EXISTS `Freelancers`;
DROP TABLE IF EXISTS `Clientes`;
-- DROP ROLE IF EXISTS 'rol_freelancer', 'rol_cliente', 'rol_admin';

-- --------------------------------------------------------------------------
-- SECCIÓN 1: Creación de Tablas y Restricciones (con mejoras)
-- --------------------------------------------------------------------------

-- Tabla: Clientes (Sin cambios mayores respecto a v2)
CREATE TABLE `Clientes` (
  `cliente_id` INT AUTO_INCREMENT PRIMARY KEY,
  `nombre_empresa` VARCHAR(150) NOT NULL,
  `nombre_contacto` VARCHAR(100) NOT NULL,
  `email` VARCHAR(100) NOT NULL UNIQUE,
  `telefono` VARCHAR(20) NULL,
  `contrasena_hash` VARCHAR(255) NULL,
  `fecha_registro` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  `fecha_modificacion` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `activo` BOOLEAN DEFAULT TRUE,
  CONSTRAINT `chk_email_cliente_formato` CHECK (`email` LIKE '_%@_%.__%')
);

-- Tabla: Freelancers (con columna generada para nombre completo)
CREATE TABLE `Freelancers` (
  `freelancer_id` INT AUTO_INCREMENT PRIMARY KEY,
  `nombre` VARCHAR(50) NOT NULL,
  `apellido` VARCHAR(50) NOT NULL,
  `nombre_completo` VARCHAR(101) AS (CONCAT(`nombre`, ' ', `apellido`)) STORED, -- Columna Generada (MySQL 5.7+/MariaDB 10.2+)
  `email` VARCHAR(100) NOT NULL UNIQUE,
  `contrasena_hash` VARCHAR(255) NOT NULL,
  `especialidad` VARCHAR(100) NULL,
  `tarifa_hora` DECIMAL(10, 2) NULL CHECK (`tarifa_hora` IS NULL OR `tarifa_hora` > 0),
  `fecha_registro` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  `fecha_modificacion` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `disponible` BOOLEAN DEFAULT TRUE,
  CONSTRAINT `chk_email_freelancer_formato` CHECK (`email` LIKE '_%@_%.__%')
);

-- Tabla: CategoriasProyectos (Sin cambios mayores respecto a v2)
CREATE TABLE `CategoriasProyectos` (
  `categoria_id` INT AUTO_INCREMENT PRIMARY KEY,
  `nombre_categoria` VARCHAR(100) NOT NULL UNIQUE,
  `descripcion` TEXT NULL,
  `fecha_creacion` TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabla: Proyectos (con JSON y Full-Text)
CREATE TABLE `Proyectos` (
  `proyecto_id` INT AUTO_INCREMENT PRIMARY KEY,
  `cliente_id` INT NOT NULL,
  `categoria_id` INT NULL,
  `titulo` VARCHAR(200) NOT NULL,
  `descripcion` TEXT NOT NULL,
  `presupuesto` DECIMAL(12, 2) NULL CHECK (`presupuesto` IS NULL OR `presupuesto` >= 0),
  `fecha_publicacion` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  `fecha_limite` DATE NULL,
  `estado` ENUM('Abierto', 'En Progreso', 'Completado', 'Cancelado', 'En Disputa') DEFAULT 'Abierto',
  `metadata` JSON NULL, -- NUEVO: Campo JSON para datos flexibles
  `fecha_creacion` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  `fecha_modificacion` TIMESTAMP DEFAULT CURRENT_TIMESTAMP, -- Se actualizará con TRIGGER de concurrencia
  `row_version` INT UNSIGNED DEFAULT 1,
  FOREIGN KEY (`cliente_id`) REFERENCES `Clientes` (`cliente_id`) ON DELETE RESTRICT ON UPDATE CASCADE,
  FOREIGN KEY (`categoria_id`) REFERENCES `CategoriasProyectos` (`categoria_id`) ON DELETE SET NULL ON UPDATE CASCADE,
  CONSTRAINT `chk_fecha_limite_proyecto` CHECK (`fecha_limite` IS NULL OR `fecha_limite` >= DATE(`fecha_publicacion`)),
  FULLTEXT KEY `idx_ft_proyecto_titulo_desc` (`titulo`, `descripcion`) -- NUEVO: Índice Full-Text
);

-- Tabla: Tareas (con Full-Text)
CREATE TABLE `Tareas` (
  `tarea_id` INT AUTO_INCREMENT PRIMARY KEY,
  `proyecto_id` INT NOT NULL,
  `titulo` VARCHAR(255) NOT NULL,
  `descripcion` TEXT NULL,
  `fecha_limite` DATE NULL,
  `prioridad` TINYINT DEFAULT 3 CHECK (`prioridad` BETWEEN 1 AND 5),
  `estado` ENUM('Pendiente', 'En Progreso', 'Completada', 'Bloqueada', 'Cancelada') DEFAULT 'Pendiente',
  `fecha_creacion` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  `fecha_modificacion` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (`proyecto_id`) REFERENCES `Proyectos` (`proyecto_id`) ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT `chk_fecha_limite_tarea` CHECK (`fecha_limite` IS NULL OR `fecha_limite` >= DATE(`fecha_creacion`)),
  FULLTEXT KEY `idx_ft_tarea_titulo_desc` (`titulo`, `descripcion`) -- NUEVO: Índice Full-Text
);

-- Tabla: AsignacionesTareas (Sin cambios mayores respecto a v2)
CREATE TABLE `AsignacionesTareas` (
  `asignacion_id` INT AUTO_INCREMENT PRIMARY KEY,
  `tarea_id` INT NOT NULL,
  `freelancer_id` INT NOT NULL,
  `fecha_asignacion` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (`tarea_id`) REFERENCES `Tareas` (`tarea_id`) ON DELETE CASCADE ON UPDATE CASCADE,
  FOREIGN KEY (`freelancer_id`) REFERENCES `Freelancers` (`freelancer_id`) ON DELETE CASCADE ON UPDATE CASCADE,
  UNIQUE (`tarea_id`, `freelancer_id`)
);

-- Tabla: Pagos (Sin cambios mayores respecto a v2)
CREATE TABLE `Pagos` (
  `pago_id` INT AUTO_INCREMENT PRIMARY KEY,
  `proyecto_id` INT NULL,
  `tarea_id` INT NULL,
  `freelancer_id` INT NOT NULL,
  `cliente_id` INT NULL,
  `monto` DECIMAL(10, 2) NOT NULL CHECK (`monto` > 0),
  `moneda` CHAR(3) DEFAULT 'USD',
  `fecha_pago` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  `metodo_pago` VARCHAR(50) NULL,
  `referencia_pago` VARCHAR(100) NULL UNIQUE,
  `estado_pago` ENUM('Pendiente', 'Completado', 'Fallido', 'Reembolsado') DEFAULT 'Completado',
  FOREIGN KEY (`proyecto_id`) REFERENCES `Proyectos` (`proyecto_id`) ON DELETE SET NULL,
  FOREIGN KEY (`tarea_id`) REFERENCES `Tareas` (`tarea_id`) ON DELETE SET NULL,
  FOREIGN KEY (`freelancer_id`) REFERENCES `Freelancers` (`freelancer_id`) ON DELETE RESTRICT,
  FOREIGN KEY (`cliente_id`) REFERENCES `Clientes` (`cliente_id`) ON DELETE SET NULL,
  CONSTRAINT `chk_pago_asociado` CHECK (`proyecto_id` IS NOT NULL OR `tarea_id` IS NOT NULL),
  CONSTRAINT `chk_pago_referencia_si_completado` CHECK (NOT (`estado_pago` = 'Completado' AND `referencia_pago` IS NULL))
);

-- NUEVO: Tabla para Auditoría Detallada
CREATE TABLE `HistorialCambios` (
  `historial_id` BIGINT AUTO_INCREMENT PRIMARY KEY,
  `tabla_afectada` VARCHAR(64) NOT NULL,
  `registro_id` INT NOT NULL, -- PK de la tabla afectada
  `campo_afectado` VARCHAR(64) NOT NULL,
  `valor_anterior` TEXT NULL,
  `valor_nuevo` TEXT NULL,
  `tipo_operacion` ENUM('INSERT', 'UPDATE', 'DELETE') NOT NULL,
  `fecha_operacion` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  `usuario_db` VARCHAR(128) DEFAULT USER(), -- Usuario de la BD que realizó la operación
  INDEX `idx_historial_tabla_registro` (`tabla_afectada`, `registro_id`, `fecha_operacion`) -- Índice para buscar historial
);

-- --------------------------------------------------------------------------
-- SECCIÓN 2: Índices para Optimización (Incluyendo Full-Text)
-- --------------------------------------------------------------------------
-- (Índices de v2 se mantienen, aquí solo se muestran los FT añadidos en CREATE TABLE)
-- idx_ft_proyecto_titulo_desc ON Proyectos(titulo, descripcion)
-- idx_ft_tarea_titulo_desc ON Tareas(titulo, descripcion)

-- (Revisar y añadir más índices compuestos si el análisis de consultas lo requiere)
CREATE INDEX `idx_freelancer_nombre_completo` ON `Freelancers` (`nombre_completo`); -- Indexar columna generada si se busca por ella


-- --------------------------------------------------------------------------
-- SECCIÓN 3: Triggers (Auditoría y Concurrencia)
-- --------------------------------------------------------------------------

-- Trigger para Concurrencia (Optimistic Locking) y actualizar fecha_modificacion
DELIMITER //
CREATE TRIGGER `trg_proyecto_before_update_concurrency`
BEFORE UPDATE ON `Proyectos`
FOR EACH ROW
BEGIN
  SET NEW.`fecha_modificacion` = CURRENT_TIMESTAMP;
  IF NEW.`row_version` != OLD.`row_version` + 1 THEN
      SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Error de concurrencia: El registro ha sido modificado.';
  END IF;
END //
DELIMITER ;

-- NUEVO: Triggers para Auditoría Detallada en la tabla Proyectos

DELIMITER //
CREATE TRIGGER `trg_proyecto_after_insert_audit`
AFTER INSERT ON `Proyectos`
FOR EACH ROW
BEGIN
    -- Registrar cada campo insertado como un cambio de NULL a valor nuevo
    INSERT INTO `HistorialCambios` (`tabla_afectada`, `registro_id`, `campo_afectado`, `valor_anterior`, `valor_nuevo`, `tipo_operacion`) VALUES
    ('Proyectos', NEW.proyecto_id, 'cliente_id', NULL, NEW.cliente_id, 'INSERT'),
    ('Proyectos', NEW.proyecto_id, 'categoria_id', NULL, NEW.categoria_id, 'INSERT'),
    ('Proyectos', NEW.proyecto_id, 'titulo', NULL, NEW.titulo, 'INSERT'),
    ('Proyectos', NEW.proyecto_id, 'descripcion', NULL, NEW.descripcion, 'INSERT'),
    ('Proyectos', NEW.proyecto_id, 'presupuesto', NULL, NEW.presupuesto, 'INSERT'),
    ('Proyectos', NEW.proyecto_id, 'fecha_limite', NULL, NEW.fecha_limite, 'INSERT'),
    ('Proyectos', NEW.proyecto_id, 'estado', NULL, NEW.estado, 'INSERT'),
    ('Proyectos', NEW.proyecto_id, 'metadata', NULL, IFNULL(CAST(NEW.metadata AS CHAR), 'NULL'), 'INSERT');
    -- Añadir más campos si es necesario
END //
DELIMITER ;

DELIMITER //
CREATE TRIGGER `trg_proyecto_after_update_audit`
AFTER UPDATE ON `Proyectos`
FOR EACH ROW
BEGIN
    -- Registrar solo los campos que realmente cambiaron
    IF OLD.cliente_id <=> NEW.cliente_id = 0 THEN -- Usar <=> para comparación segura con NULLs
        INSERT INTO `HistorialCambios` (`tabla_afectada`, `registro_id`, `campo_afectado`, `valor_anterior`, `valor_nuevo`, `tipo_operacion`)
        VALUES ('Proyectos', OLD.proyecto_id, 'cliente_id', OLD.cliente_id, NEW.cliente_id, 'UPDATE');
    END IF;
    IF OLD.categoria_id <=> NEW.categoria_id = 0 THEN
        INSERT INTO `HistorialCambios` (`tabla_afectada`, `registro_id`, `campo_afectado`, `valor_anterior`, `valor_nuevo`, `tipo_operacion`)
        VALUES ('Proyectos', OLD.proyecto_id, 'categoria_id', OLD.categoria_id, NEW.categoria_id, 'UPDATE');
    END IF;
    IF OLD.titulo <=> NEW.titulo = 0 THEN
        INSERT INTO `HistorialCambios` (`tabla_afectada`, `registro_id`, `campo_afectado`, `valor_anterior`, `valor_nuevo`, `tipo_operacion`)
        VALUES ('Proyectos', OLD.proyecto_id, 'titulo', OLD.titulo, NEW.titulo, 'UPDATE');
    END IF;
     IF OLD.descripcion <=> NEW.descripcion = 0 THEN
        INSERT INTO `HistorialCambios` (`tabla_afectada`, `registro_id`, `campo_afectado`, `valor_anterior`, `valor_nuevo`, `tipo_operacion`)
        VALUES ('Proyectos', OLD.proyecto_id, 'descripcion', SUBSTRING(OLD.descripcion, 1, 500), SUBSTRING(NEW.descripcion, 1, 500), 'UPDATE'); -- Limitar longitud para TEXT
    END IF;
    IF OLD.presupuesto <=> NEW.presupuesto = 0 THEN
        INSERT INTO `HistorialCambios` (`tabla_afectada`, `registro_id`, `campo_afectado`, `valor_anterior`, `valor_nuevo`, `tipo_operacion`)
        VALUES ('Proyectos', OLD.proyecto_id, 'presupuesto', OLD.presupuesto, NEW.presupuesto, 'UPDATE');
    END IF;
    IF OLD.fecha_limite <=> NEW.fecha_limite = 0 THEN
        INSERT INTO `HistorialCambios` (`tabla_afectada`, `registro_id`, `campo_afectado`, `valor_anterior`, `valor_nuevo`, `tipo_operacion`)
        VALUES ('Proyectos', OLD.proyecto_id, 'fecha_limite', OLD.fecha_limite, NEW.fecha_limite, 'UPDATE');
    END IF;
    IF OLD.estado <=> NEW.estado = 0 THEN
        INSERT INTO `HistorialCambios` (`tabla_afectada`, `registro_id`, `campo_afectado`, `valor_anterior`, `valor_nuevo`, `tipo_operacion`)
        VALUES ('Proyectos', OLD.proyecto_id, 'estado', OLD.estado, NEW.estado, 'UPDATE');
    END IF;
    IF OLD.metadata <=> NEW.metadata = 0 THEN -- JSON requiere comparación cuidadosa
        INSERT INTO `HistorialCambios` (`tabla_afectada`, `registro_id`, `campo_afectado`, `valor_anterior`, `valor_nuevo`, `tipo_operacion`)
        VALUES ('Proyectos', OLD.proyecto_id, 'metadata', IFNULL(CAST(OLD.metadata AS CHAR), 'NULL'), IFNULL(CAST(NEW.metadata AS CHAR), 'NULL'), 'UPDATE');
    END IF;
    -- Añadir más campos si es necesario
END //
DELIMITER ;


DELIMITER //
CREATE TRIGGER `trg_proyecto_after_delete_audit`
AFTER DELETE ON `Proyectos`
FOR EACH ROW
BEGIN
    -- Registrar la eliminación del registro
    INSERT INTO `HistorialCambios` (`tabla_afectada`, `registro_id`, `campo_afectado`, `valor_anterior`, `valor_nuevo`, `tipo_operacion`)
    VALUES ('Proyectos', OLD.proyecto_id, 'REGISTRO_COMPLETO', CONCAT('Titulo: ', OLD.titulo), NULL, 'DELETE');
END //
DELIMITER ;


-- --------------------------------------------------------------------------
-- SECCIÓN 4: Vistas (Sin cambios mayores respecto a v2)
-- --------------------------------------------------------------------------
CREATE OR REPLACE VIEW `vista_proyectos_activos_detalle` AS
SELECT
    p.proyecto_id,
    p.titulo AS titulo_proyecto,
    p.estado AS estado_proyecto,
    p.presupuesto,
    p.fecha_publicacion,
    p.fecha_limite,
    cl.nombre_empresa AS cliente,
    cl.email AS email_cliente,
    cp.nombre_categoria AS categoria,
    p.metadata, -- Incluir campo JSON en la vista
    p.row_version
FROM
    `Proyectos` p
JOIN
    `Clientes` cl ON p.cliente_id = cl.cliente_id
LEFT JOIN
    `CategoriasProyectos` cp ON p.categoria_id = cp.categoria_id
WHERE
    p.estado IN ('Abierto', 'En Progreso', 'En Disputa');

-- --------------------------------------------------------------------------
-- SECCIÓN 5: Procedimientos y Funciones Almacenadas
-- --------------------------------------------------------------------------

-- NUEVO: Función para obtener nombre completo
DELIMITER //
CREATE FUNCTION `fn_NombreCompletoFreelancer` (p_freelancer_id INT)
RETURNS VARCHAR(101)
DETERMINISTIC -- Importante si no accede a tablas o usa valores fijos
READS SQL DATA -- Indica que lee datos
BEGIN
    DECLARE v_nombre_completo VARCHAR(101);
    SELECT CONCAT(nombre, ' ', apellido) INTO v_nombre_completo
    FROM Freelancers
    WHERE freelancer_id = p_freelancer_id;
    RETURN IFNULL(v_nombre_completo, 'Desconocido');
END //
DELIMITER ;

-- NUEVO: Procedimiento para asignar tarea a freelancer con validaciones
DELIMITER //
CREATE PROCEDURE `sp_AsignarFreelancerTarea` (
    IN p_tarea_id INT,
    IN p_freelancer_id INT,
    OUT p_resultado VARCHAR(255) -- Parámetro de salida para mensaje
)
BEGIN
    DECLARE v_tarea_existe INT DEFAULT 0;
    DECLARE v_freelancer_existe INT DEFAULT 0;
    DECLARE v_freelancer_disponible BOOLEAN DEFAULT FALSE;
    DECLARE v_tarea_asignable BOOLEAN DEFAULT FALSE;
    DECLARE v_ya_asignado INT DEFAULT 0;

    -- Validar existencia de tarea y freelancer
    SELECT COUNT(*) INTO v_tarea_existe FROM Tareas WHERE tarea_id = p_tarea_id;
    SELECT COUNT(*), disponible INTO v_freelancer_existe, v_freelancer_disponible FROM Freelancers WHERE freelancer_id = p_freelancer_id;

    IF v_tarea_existe = 0 THEN
        SET p_resultado = 'Error: La tarea especificada no existe.';
    ELSEIF v_freelancer_existe = 0 THEN
        SET p_resultado = 'Error: El freelancer especificado no existe.';
    ELSE
        -- Validar si la tarea está en un estado asignable (ej. no 'Completada' o 'Cancelada')
        SELECT (estado IN ('Pendiente', 'En Progreso', 'Bloqueada')) INTO v_tarea_asignable
        FROM Tareas WHERE tarea_id = p_tarea_id;

        -- Validar si ya existe la asignación
        SELECT COUNT(*) INTO v_ya_asignado FROM AsignacionesTareas
        WHERE tarea_id = p_tarea_id AND freelancer_id = p_freelancer_id;

        IF NOT v_freelancer_disponible THEN
             SET p_resultado = CONCAT('Advertencia: El freelancer ', fn_NombreCompletoFreelancer(p_freelancer_id), ' no está marcado como disponible, pero se asignará.');
             -- Podría hacerse un SIGNAL SQLSTATE para error si no se permite asignar a no disponibles
        END IF;

        IF NOT v_tarea_asignable THEN
             SET p_resultado = 'Error: La tarea no está en un estado que permita asignación.';
        ELSEIF v_ya_asignado > 0 THEN
             SET p_resultado = 'Error: El freelancer ya está asignado a esta tarea.';
        ELSE
            -- Realizar la inserción
            INSERT INTO AsignacionesTareas (tarea_id, freelancer_id) VALUES (p_tarea_id, p_freelancer_id);
            SET p_resultado = CONCAT('Éxito: Freelancer ', fn_NombreCompletoFreelancer(p_freelancer_id), ' asignado a la tarea ID ', p_tarea_id, '. ', IFNULL(p_resultado,'')); -- Añadir advertencia si existe
        END IF;
    END IF;
END //
DELIMITER ;


-- --------------------------------------------------------------------------
-- SECCIÓN 6: Seguridad Básica (Roles y Permisos - Mismos ejemplos de v2)
-- --------------------------------------------------------------------------
/* -- Descomentar para ejecutar (requiere privilegios adecuados)
CREATE ROLE IF NOT EXISTS 'rol_freelancer', 'rol_cliente', 'rol_admin';
GRANT SELECT ON `FreelanceProManager`.* TO 'rol_freelancer'; -- Ajustar permisos más finos
GRANT EXECUTE ON FUNCTION `FreelanceProManager`.`fn_NombreCompletoFreelancer` TO 'rol_freelancer', 'rol_cliente', 'rol_admin';
GRANT EXECUTE ON PROCEDURE `FreelanceProManager`.`sp_AsignarFreelancerTarea` TO 'rol_cliente', 'rol_admin'; -- Clientes o Admins pueden asignar
-- ... (Resto de GRANTs y asignación de usuarios) ...
FLUSH PRIVILEGES;
*/

-- --------------------------------------------------------------------------
-- SECCIÓN 7: Inserción de Datos de Ejemplo (Añadiendo metadata JSON)
-- --------------------------------------------------------------------------
-- (Clientes, Freelancers, Categorías sin cambios respecto a v2)
INSERT INTO `Clientes` (`nombre_empresa`, `nombre_contacto`, `email`, `telefono`, `contrasena_hash`, `activo`) VALUES
('Innovatech Global', 'Elena Rodríguez', 'elena.r@innovatech.global', '555-1122', '$2b$12$placeholderhashcliente1...', TRUE),
('Creative Solutions Hub', 'David Gómez', 'david.g@creativesolutions.hub', '555-3344', '$2b$12$placeholderhashcliente2...', TRUE);
INSERT INTO `Freelancers` (`nombre`, `apellido`, `email`, `contrasena_hash`, `especialidad`, `tarifa_hora`, `disponible`) VALUES
('Carlos', 'Silva', 'carlos.silva@email.freelance', '$2b$12$placeholderhashfreelancer1...', 'Desarrollo Full-Stack', 70.00, TRUE),
('Laura', 'Méndez', 'laura.mendez@email.freelance', '$2b$12$placeholderhashfreelancer2...', 'Marketing de Contenidos', 45.00, TRUE),
('Andrés', 'Vega', 'andres.vega@email.freelance', '$2b$12$placeholderhashfreelancer3...', 'Diseño Gráfico', 55.00, FALSE);
INSERT INTO `CategoriasProyectos` (`nombre_categoria`, `descripcion`) VALUES
('Desarrollo Web', 'Creación de sitios y aplicaciones web.'),
('Marketing y Ventas', 'Estrategias de promoción, SEO, SEM, gestión de redes sociales.'),
('Diseño e Identidad Visual', 'Logotipos, branding, material gráfico.');

-- Proyectos con metadata
INSERT INTO `Proyectos` (`cliente_id`, `categoria_id`, `titulo`, `descripcion`, `presupuesto`, `fecha_limite`, `estado`, `metadata`, `row_version`) VALUES
(1, 1, 'Plataforma E-learning Interactiva', 'Desarrollar una plataforma Moodle personalizada con gamificación.', 12000.00, '2025-12-01', 'Abierto', '{"tecnologias_preferidas": ["PHP", "React"], "integraciones": ["PayPal", "Zoom"]}', 1),
(2, 3, 'Rediseño Completo de Marca', 'Actualizar logotipo, paleta de colores y guía de estilo.', 4500.00, '2025-08-31', 'En Progreso', '{"entregables_clave": ["Logo Vectorial", "Manual de Marca PDF"], "competencia_ref": ["Competidor A", "Competidor B"]}', 1),
(1, 2, 'Campaña SEO para Blog Corporativo', 'Optimización on-page y estrategia de link building.', 2500.00, NULL, 'En Progreso', '{"keywords_objetivo": ["software a medida", "consultoría tech"], "region_enfoque": "LATAM"}', 1);

-- (Tareas, Asignaciones, Pagos sin cambios respecto a v2)
INSERT INTO `Tareas` (`proyecto_id`, `titulo`, `descripcion`, `fecha_limite`, `prioridad`, `estado`) VALUES
(1, 'Diseño de Base de Datos Usuarios y Cursos', 'Crear el esquema E-R y script SQL.', '2025-06-15', 1, 'Pendiente'),(1, 'Implementación Módulo de Autenticación', 'Login, registro, recuperación de contraseña.', '2025-07-10', 1, 'Pendiente'),(2, 'Investigación y Moodboard', 'Recopilar inspiración y definir dirección visual.', '2025-05-30', 2, 'Completada'),(2, 'Diseño de Propuestas de Logotipo', 'Crear 3 variaciones iniciales.', '2025-06-20', 1, 'En Progreso'),(3, 'Auditoría SEO Técnica del Blog', 'Identificar problemas técnicos actuales.', '2025-06-05', 2, 'En Progreso');
INSERT INTO `AsignacionesTareas` (`tarea_id`, `freelancer_id`) VALUES (1, 1),(2, 1),(3, 3),(4, 3),(5, 2);
INSERT INTO `Pagos` (`proyecto_id`, `tarea_id`, `freelancer_id`, `cliente_id`, `monto`, `moneda`, `metodo_pago`, `referencia_pago`, `estado_pago`) VALUES (2, 3, 3, 2, 300.00, 'USD', 'Transferencia', 'PAGO-T3-REDIS-AVEGA', 'Completado'),(1, NULL, 1, 1, 2000.00, 'USD', 'Stripe', 'STRIPE-ANTICIPO-P1-CSILVA', 'Completado');

-- --------------------------------------------------------------------------
-- SECCIÓN 8: Ejemplos de Uso de Funcionalidades Avanzadas
-- --------------------------------------------------------------------------

/*
-- Ejemplo: Usar la función de nombre completo
SELECT fn_NombreCompletoFreelancer(1); -- Devuelve 'Carlos Silva'

-- Ejemplo: Usar el procedimiento almacenado para asignar tarea 1 al freelancer 2
CALL sp_AsignarFreelancerTarea(1, 2, @resultado);
SELECT @resultado; -- Muestra el mensaje de éxito o error

-- Ejemplo: Búsqueda Full-Text en proyectos por "plataforma interactiva"
SELECT proyecto_id, titulo, MATCH(titulo, descripcion) AGAINST ('plataforma interactiva' IN NATURAL LANGUAGE MODE) AS score
FROM Proyectos
WHERE MATCH(titulo, descripcion) AGAINST ('plataforma interactiva' IN NATURAL LANGUAGE MODE)
ORDER BY score DESC;

-- Ejemplo: Extraer datos del campo JSON (MySQL 5.7+)
SELECT proyecto_id, titulo, JSON_EXTRACT(metadata, '$.tecnologias_preferidas[0]') AS primera_tecnologia
FROM Proyectos
WHERE JSON_CONTAINS(metadata, '["React"]', '$.tecnologias_preferidas'); -- Proyectos que prefieren React

-- Ejemplo: Ver el historial de cambios del proyecto con ID 2
SELECT * FROM HistorialCambios
WHERE tabla_afectada = 'Proyectos' AND registro_id = 2
ORDER BY fecha_operacion DESC;

-- Ejemplo: Consulta con Window Function para Ranking de Freelancers por Tarifa por Hora dentro de su especialidad
SELECT
    freelancer_id,
    nombre_completo,
    especialidad,
    tarifa_hora,
    RANK() OVER (PARTITION BY especialidad ORDER BY tarifa_hora DESC) as ranking_tarifa_especialidad
FROM Freelancers
WHERE especialidad IS NOT NULL AND tarifa_hora IS NOT NULL;

*/

-- --------------------------------------------------------------------------
-- FIN DEL SCRIPT v3
-- --------------------------------------------------------------------------
