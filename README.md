-- Tabla Cliente
CREATE TABLE cliente (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    correo VARCHAR(100),
    tipo VARCHAR(50) -- estudiante, profesor, personal, externo
);

-- Tabla Producto
CREATE TABLE producto (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    descripcion TEXT,
    precio DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL DEFAULT 0
);

-- Tabla Usuario
CREATE TABLE usuario (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    rol VARCHAR(50)
);

-- Tabla Pedido
CREATE TABLE pedido (
    id INT AUTO_INCREMENT PRIMARY KEY,
    fecha DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    id_cliente INT,
    estado VARCHAR(50),
    FOREIGN KEY (id_cliente) REFERENCES cliente(id)
);

-- Tabla DetallePedido
CREATE TABLE detalle_pedido (
    id INT AUTO_INCREMENT PRIMARY KEY,
    id_pedido INT,
    id_producto INT,
    id_usuario INT,
    cantidad INT NOT NULL,
    precio_unitario DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (id_pedido) REFERENCES pedido(id) ON DELETE CASCADE,
    FOREIGN KEY (id_producto) REFERENCES producto(id),
    FOREIGN KEY (id_usuario) REFERENCES usuario(id)
);

-- Tabla Venta
CREATE TABLE venta (
    id INT AUTO_INCREMENT PRIMARY KEY,
    id_pedido INT,
    id_usuario INT,
    fecha DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    total DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (id_pedido) REFERENCES pedido(id),
    FOREIGN KEY (id_usuario) REFERENCES usuario(id)
);

-- Tabla Auditoria
CREATE TABLE reporte (
    id INT AUTO_INCREMENT PRIMARY KEY,
    id_usuario INT, -- usuario que realiza la acción (puede ser NULL si no está disponible)
    accion VARCHAR(100) NOT NULL, -- descripción breve de la acción
    fecha DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    tabla_afectada VARCHAR(50) NOT NULL, -- nombre de la tabla impactada
    id_registro INT, -- id del registro afectado
    id_producto INT, -- producto involucrado (si aplica)
    cantidad_antes INT, -- cantidad anterior (por ejemplo, stock antes del cambio)
    cantidad_despues INT, -- cantidad después del cambio
    cantidad_movida INT, -- diferencia de cantidad (por ejemplo, cantidad vendida, ajustada)
    detalle TEXT, -- información adicional (puede ser texto o JSON)
    FOREIGN KEY (id_usuario) REFERENCES usuario(id),
    FOREIGN KEY (id_producto) REFERENCES producto(id)
);

-- trigger para actualizar stock en producto
DELIMITER $$
CREATE TRIGGER descontar_stock_producto
AFTER INSERT ON detalle_pedido
FOR EACH ROW
BEGIN
    UPDATE producto
    SET stock = stock - NEW.cantidad
    WHERE id = NEW.id_producto;
END$$
DELIMITER ;

-- trigger para registros de auditorias al realizar  una venta
DELIMITER $$
CREATE TRIGGER reportar_venta
AFTER INSERT ON venta
FOR EACH ROW
BEGIN
    /* Trigger reportar_venta -> registra en la tabla 'reporte' cada venta realizada.
     Por cada producto vendido en el pedido, se inserta un registro con información relevante.
     Utiliza un cursor para recorrer los detalles del pedido asociado a la venta.
     Guarda el stock antes y después de la venta, la cantidad vendida y otros datos útiles para reportes y auditoría.
     Permite rastrear el movimiento de inventario y el historial de ventas por producto y usuario.
     */
    DECLARE done INT DEFAULT FALSE; -- DECLARE para definir variables locales dentro de procedimientos almacenados en SQL.
    -- En este caso, 'done' es una variable de tipo INT que se inicializa en FALSE (0) es una badera para controlar el flujo.
    DECLARE dp_id INT;
    DECLARE dp_producto INT;
    DECLARE dp_cantidad INT;
    DECLARE dp_precio DECIMAL(10,2);
    DECLARE stock_antes INT;
    DECLARE stock_despues INT;
    
    -- Cursor para recorrer cada detalle de pedido involucrado en la venta
    DECLARE cur CURSOR FOR
        SELECT id, id_producto, cantidad, precio_unitario
        FROM detalle_pedido
        WHERE id_pedido = NEW.id_pedido;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    OPEN cur;
        read_loop: LOOP  
            -- El bucle LOOP recorre cada registro del cursor 'cur', que contiene los detalles del pedido asociado a la venta.
            FETCH cur INTO dp_id, dp_producto, dp_cantidad, dp_precio; -- Extrae los datos de cada detalle de pedido.
            IF done THEN
                -- Si no hay más registros, sale del bucle.
                LEAVE read_loop;
            END IF;
            -- Obtiene el stock del producto antes de la venta para registrar el movimiento en auditoría.
            SELECT stock INTO stock_antes FROM producto WHERE id = dp_producto;
            -- Calcula el stock después de la venta.
            SET stock_despues = stock_antes - dp_cantidad;
            -- Inserta un registro en la tabla 'reporte' con toda la información relevante del movimiento de inventario y venta.
            INSERT INTO reporte (
                id_usuario,
                accion,
                fecha,
                tabla_afectada,
                id_registro,
                id_producto,
                cantidad_antes,
                cantidad_despues,
                cantidad_movida,
                detalle
            )
            VALUES (
                NEW.id_usuario,
                'Registro de venta',
            NOW(),
            'venta',
            NEW.id,
            dp_producto,
            stock_antes,
            stock_despues,
            dp_cantidad,
            CONCAT('Venta de producto ', dp_producto, ': cantidad vendida=', dp_cantidad, ', precio_unitario=', dp_precio)
        );
    END LOOP;
    CLOSE cur;
END$$
DELIMITER ;

-- trigger para reponer stock en caso de que se elimine un detalle de pedido
DELIMITER $$
CREATE TRIGGER reponer_stock_producto
AFTER DELETE ON detalle_pedido
FOR EACH ROW
BEGIN
    UPDATE producto
    SET stock = stock + OLD.cantidad
    WHERE id = OLD.id_producto;
END$$
DELIMITER ;

-- trigger de para registro de auditoria al haberse realizado un cambio en producto
DELIMITER $$
CREATE TRIGGER reportar_movimiento_inventario
AFTER UPDATE ON producto
FOR EACH ROW
BEGIN
    IF NEW.stock <> OLD.stock THEN
        INSERT INTO reporte (
            id_usuario, 
            accion, 
            fecha, 
            tabla_afectada, 
            id_registro, 
            id_producto,
            cantidad_antes,
            cantidad_despues,
            cantidad_movida,
            detalle
        ) VALUES (
            NULL, -- de momento al no tener una app que almacene el id_usuario pasamos null a hacer movimiento de inventario
            'Actualización de inventario',
            NOW(),
            'producto',
            NEW.id,
            NEW.id,
            OLD.stock,
            NEW.stock,
            NEW.stock - OLD.stock,
            CONCAT('Cambio de stock en producto ', NEW.nombre, 
                   '. Antes: ', OLD.stock, 
                   ', Después: ', NEW.stock)
        );
    END IF;
END$$
DELIMITER ;


-- insertemos unos datos de prueba

-- Clientes
INSERT INTO cliente (nombre, correo, tipo) VALUES 
('Ana Torres', 'ana.torres@uni.edu', 'estudiante'),
('Carlos Rivas', 'carlos.rivas@uni.edu', 'profesor'),
('Luisa Gómez', 'luisa.gomez@uni.edu', 'personal');

-- Usuarios
INSERT INTO usuario (nombre, rol) VALUES
('María López', 'cajero'),
('Juan Pérez', 'admin');

-- Productos
INSERT INTO producto (nombre, descripcion, precio, stock) VALUES
('Café Americano', 'Taza de café clásico', 1.50, 100),
('Sandwich', 'Sandwich de jamón y queso', 2.00, 50),
('Jugo de Naranja', 'Vaso de jugo natural', 1.20, 80);

-- Pedidos
INSERT INTO pedido (fecha, id_cliente, estado) VALUES
(NOW(), 1, 'pendiente'),
(NOW(), 2, 'pagado'),
(NOW(), 3, 'cancelado');

-- Detalle de Pedidos
INSERT INTO detalle_pedido (id_usuario,id_pedido, id_producto, cantidad, precio_unitario) VALUES
(1,1, 1, 2, 1.50), -- Pedido 1: 2 Café Americano
(1,1, 2, 1, 2.00), -- Pedido 1: 1 Sandwich
(1,2, 3, 3, 1.20); -- Pedido 2: 3 Jugos de Naranja

-- Ventas
INSERT INTO venta (id_pedido, id_usuario, fecha, total) VALUES
(2, 1, NOW(), 3.60);


-- consultas avanzadas

-- Historial de ventas por producto

SELECT 
    r.fecha,
    p.nombre AS producto,
    r.cantidad_movida AS cantidad_vendida,
    r.cantidad_antes,
    r.cantidad_despues,
    u.nombre AS usuario,
    r.detalle
FROM
    reporte r
JOIN producto p ON r.id_producto = p.id
LEFT JOIN usuario u ON r.id_usuario = u.id
WHERE
    r.accion = 'Registro de venta'
ORDER BY
    r.fecha DESC;
    
    -- ranking de productos con mayor cantidad vendida
    SELECT
    p.nombre AS producto,
    SUM(r.cantidad_movida) AS total_vendida -- suma total
FROM
    reporte r
JOIN producto p ON r.id_producto = p.id
WHERE
    r.accion = 'Registro de venta'
GROUP BY
    r.id_producto, p.nombre
ORDER BY
    total_vendida DESC
LIMIT 10;

-- movimiento de inventario por producto
SELECT
    r.fecha,
    p.nombre AS producto,
    r.cantidad_antes,
    r.cantidad_despues,
    r.cantidad_movida,
    r.detalle
FROM
    reporte r
JOIN producto p ON r.id_producto = p.id
WHERE
    r.accion = 'Actualización de inventario'
ORDER BY
    r.fecha DESC;

-- consumo total por usuario - ventas hechas
SELECT
    u.nombre AS usuario,
    SUM(r.cantidad_movida) AS total_productos_vendidos
FROM
    reporte r
JOIN usuario u ON r.id_usuario = u.id
WHERE
    r.accion = 'Registro de venta'
GROUP BY
    r.id_usuario, u.nombre
ORDER BY
    total_productos_vendidos DESC;
    
    
-- resumen de reportes por tipo de accion
SELECT
    accion,
    COUNT(*) AS cantidad_acciones
FROM
    reporte
GROUP BY
    accion
ORDER BY
    cantidad_acciones DESC;
    
-- stock actual de productos y ultimos movimientos
SELECT
    p.nombre,
    p.stock,
    (SELECT r.fecha FROM reporte r WHERE r.id_producto = p.id ORDER BY r.fecha DESC LIMIT 1) AS ultima_modificacion,
    (SELECT r.cantidad_movida FROM reporte r WHERE r.id_producto = p.id ORDER BY r.fecha DESC LIMIT 1) AS ultimo_movimiento
FROM
    producto p
ORDER BY
    p.stock ASC;
