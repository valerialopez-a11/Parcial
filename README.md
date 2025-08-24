#  Proyecto Cafetería – Gestión y Reportes

##  Autores
- **Valeria López**
- **Cristian ortiz**

---

## Descripción del Proyecto
Este proyecto implementa un **sistema de gestión para la cafetería universitaria**, diseñado para reemplazar el manejo manual en papel que generaba:

-  Pedidos perdidos  
-  Inventarios inexactos  
-  Dificultad para generar reportes de ventas  

La solución propuesta permite administrar **clientes, productos, pedidos, ventas y usuarios**, garantizando un control más preciso y confiable.  
Además, mediante **triggers y reportes avanzados**, el sistema gestiona automáticamente el stock y genera auditorías para un seguimiento detallado de las operaciones.

---

## Estructura del Proyecto

###  Tablas principales
- **Cliente** → Información de los clientes (estudiantes, profesores, personal, externos).  
- **Producto** → Productos ofrecidos en la cafetería con precio y stock.  
- **Usuario** → Usuarios del sistema (cajeros, administradores, etc.).  
- **Pedido** → Registro de pedidos generados.  
- **DetallePedido** → Información de cada producto dentro de un pedido.  
- **Venta** → Registro de ventas asociadas a pedidos.  
- **Reporte (auditoría)** → Control de ventas y movimientos de inventario.  

###  Triggers implementados
- `descontar_stock_producto` → Descuenta automáticamente el stock al registrar un pedido.  
- `reportar_venta` → Inserta en auditoría un registro de cada venta.  
- `reponer_stock_producto` → Restaura stock al eliminar un detalle de pedido.  
- `reportar_movimiento_inventario` → Audita actualizaciones de inventario en productos.  

---

## Objetivos del Proyecto
✔ Digitalizar la gestión de la cafetería.  
✔ Evitar errores en inventarios.  
✔ Automatizar movimientos de ventas y stock.  
✔ Facilitar reportes avanzados de consumo y ventas.  

---

## Ejecución

1. Crear la base de datos en **MySQL**.  
2. Ejecutar el script `cafeteria.sql` para generar las tablas y triggers.  
3. Insertar datos de prueba con los **INSERT** provistos.  
4. Ejecutar las consultas avanzadas para obtener reportes:

   - 📊 Historial de ventas por producto  
   - 🏆 Ranking de productos más vendidos  
   - 📦 Movimiento de inventario  
   - 👤 Consumo total por usuario  
   - 📑 Resumen de acciones registradas  
   - 📉 Stock actual y últimos movimientos  

---

##  Consultas Avanzadas (Ejemplos)

**Historial de ventas por producto**
```sql
SELECT r.fecha, p.nombre AS producto, r.cantidad_movida, r.cantidad_antes, r.cantidad_despues, u.nombre AS usuario, r.detalle
FROM reporte r
JOIN producto p ON r.id_producto = p.id
LEFT JOIN usuario u ON r.id_usuario = u.id
WHERE r.accion = 'Registro de venta'
ORDER BY r.fecha DESC;
