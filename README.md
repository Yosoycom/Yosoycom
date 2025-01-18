- 👋 Hi, I’m @Yosoycom
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...

<!---
Yosoycom/Yosoycom is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
-- Crear base de datos
create database if NOT exists gestion_inventario_ultra;
use gestion_inventario_ultra;

-- Crear tablas principales

-- Tabla de categorías
create table categorias (
    id_categoria INT auto_increment primary KEY,
    nombre_categoria VARCHAR(100) NOT NULL,
    descripcion TEXT
);

-- Tabla de productos
create table productos (
    id_producto INT AUTO_INCREMENT PRIMARY key,
    nombre_producto VARCHAR(150) NOT NULL,
    descripcion text,
    precio DECIMAL(10, 2) NOT NULL,
    stock INT NOT NULL DEFAULT 0,
    id_categoria INT,
    fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    foreign key (id_categoria) REFERENCES categorias(id_categoria) ON DELETE CASCADE
);

-- Tabla de proveedores
create table proveedores (
    id_proveedor INT AUTO_INCREMENT PRIMARY KEY,
    nombre_proveedor VARCHAR(150) NOT NULL,
    telefono VARCHAR(15),
    email VARCHAR(100),
    direccion TEXT
);

-- Tabla de compras
create table compras (
    id_compra INT AUTO_INCREMENT PRIMARY KEY,
    id_producto INT NOT NULL,
    id_proveedor INT NOT NULL,
    cantidad INT NOT NULL,
    costo_total DECIMAL(10, 2) NOT NULL,
    fecha_compra TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto) ON DELETE CASCADE,
    FOREIGN KEY (id_proveedor) REFERENCES proveedores(id_proveedor) ON DELETE CASCADE
);

-- Tabla de ventas
create table ventas (
    id_venta INT AUTO_INCREMENT PRIMARY KEY,
    id_producto INT NOT NULL,
    cantidad INT NOT NULL,
    precio_total DECIMAL(10, 2) NOT NULL,
    fecha_venta TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto) ON DELETE CASCADE
);

-- Tabla de movimientos de stock
create table movimientos_stock (
    id_movimiento INT AUTO_INCREMENT PRIMARY KEY,
    id_producto INT NOT NULL,
    tipo_movimiento ENUM('entrada', 'salida') NOT NULL,
    cantidad INT NOT NULL,
    fecha_movimiento TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (id_producto) REFERENCES productos(id_producto) ON DELETE CASCADE
);

-- Tabla de auditorías (registro de cambios importantes)
create table auditorias (
    id_auditoria INT AUTO_INCREMENT PRIMARY KEY,
    tabla_afectada VARCHAR(50) NOT NULL,
    operacion VARCHAR(20) NOT NULL,
    id_registro_afectado INT NOT NULL,
    detalles TEXT,
    fecha TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Crear triggers para automatización avanzada

-- Actualizar stock automáticamente al insertar en movimientos_stock
delimiter $$
create trigger after_movimiento_stock
after INSERT ON movimientos_stock
for each row
BEGIN
    IF new.tipo_movimiento = 'entrada' THEN
        update productos
        set stock = stock + new.cantidad
        where id_producto = new.id_producto;
    ELSEIF new.tipo_movimiento = 'salida' THEN
        update productos
        set stock = stock - new.cantidad
        where id_producto = new.id_producto;
    END IF;

    -- Registrar en auditorías
    INSERT INTO auditorias (tabla_afectada, operacion, id_registro_afectado, detalles)
    VALUES ('movimientos_stock', 'INSERT', new.id_movimiento, CONCAT('Tipo: ', new.tipo_movimiento, ', Cantidad: ', new.cantidad));
END$$
delimiter ;

-- Registrar automáticamente movimientos de stock al realizar una venta
delimiter $$
create trigger after_venta
after INSERT ON ventas
for each row
BEGIN
    insert into movimientos_stock (id_producto, tipo_movimiento, cantidad)
    values (new.id_producto, 'salida', new.cantidad);

    -- Registrar en auditorías
    insert into auditorias (tabla_afectada, operacion, id_registro_afectado, detalles)
    values ('ventas', 'INSERT', new.id_venta, CONCAT('Venta de ', new.cantidad, ' unidades.'));
END$$
delimiter ;

-- Registrar automáticamente movimientos de stock al realizar una compra
delimiter $$
create trigger after_compra
after INSERT ON compras
for each row
BEGIN
    insert into movimientos_stock (id_producto, tipo_movimiento, cantidad)
    values (new.id_producto, 'entrada', new.cantidad);

    -- Registrar en auditorías
    insert into auditorias (tabla_afectada, operacion, id_registro_afectado, detalles)
    values ('compras', 'INSERT', new.id_compra, CONCAT('Compra de ', new.cantidad, ' unidades de proveedor ID: ', new.id_proveedor));
END$$
delimiter ;

-- Registrar cambios en stock manualmente
delimiter $$
create trigger after_stock_update
after UPDATE ON productos
for each row
BEGIN
    IF new.stock != old.stock THEN
        insert into auditorias (tabla_afectada, operacion, id_registro_afectado, detalles)
        values ('productos', 'UPDATE', new.id_producto, CONCAT('Stock cambiado de ', old.stock, ' a ', new.stock));
    END IF;
END$$
delimiter ;

-- Insertar datos iniciales

-- Insertar categorías
insert into categorias (nombre_categoria, descripcion) values 
('Electrónica', 'Dispositivos electrónicos como televisores y teléfonos.'),
('Hogar', 'Productos para el hogar como muebles y electrodomésticos.'),
('Alimentos', 'Productos de consumo diario.');

-- Insertar productos
insert into productos (nombre_producto, descripcion, precio, stock, id_categoria) values 
('Televisor LED 40 pulgadas', 'Televisor de alta definición.', 350.99, 10, 1),
('Sofá 3 plazas', 'Sofá cómodo y elegante.', 450.50, 5, 2),
('Arroz 1kg', 'Paquete de arroz premium.', 2.99, 50, 3);

-- Insertar proveedores
insert into proveedores (nombre_proveedor, telefono, email, direccion) values 
('Proveedor Electrónica', '555-1234', 'contacto@electronica.com', 'Calle 1, Ciudad X'),
('Proveedor Hogar', '555-5678', 'soporte@hogar.com', 'Calle 2, Ciudad Y'),
('Proveedor Alimentos', '555-9876', 'ventas@alimentos.com', 'Calle 3, Ciudad Z');

-- Insertar compras
insert into compras (id_producto, id_proveedor, cantidad, costo_total) values 
(1, 1, 5, 1750.00),
(2, 2, 3, 1351.50);

-- Insertar ventas
insert into ventas (id_producto, cantidad, precio_total) values 
(1, 2, 701.98),
(3, 10, 29.90);

-- Consultas avanzadas

-- Reporte de inventario actual
select 
    p.nombre_producto, 
    p.stock, 
    c.nombre_categoria, 
    p.precio 
from 
    productos p 
join 
    categorias c on p.id_categoria = c.id_categoria;

-- Historial de movimientos de stock
select 
    m.id_movimiento, 
    p.nombre_producto, 
    m.tipo_movimiento, 
    m.cantidad, 
    m.fecha_movimiento 
from 
    movimientos_stock m 
join 
    productos p on m.id_producto = p.id_producto;

-- Historial de auditorías
select * from auditorias;
