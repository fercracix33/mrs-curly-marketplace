---
name: mrs-curly-inventario
description: "Consultar existencias y proveedores o registrar entradas, consumos, mermas y ajustes de inventario en Mrs Curly."
---

# Inventario

Aplica [el contrato común](references/operacion.md). Usa `inventory_search` con filtros para alertas o productos concretos; abre `product_get` solo para el detalle necesario. `suppliers_list` y `supplier_get` permiten resolver proveedores sin inventar IDs.

Las cantidades `Milliunits` son milésimas: 1500 significa 1,5 unidades del producto. Comprueba su unidad antes de interpretar botellas, gramos o mililitros. Una alerta no autoriza comprar ni ajustar stock.

Para una entrada usa `stock_receive`; para otros movimientos usa `stock_movement_register` con la clase que corresponda. Consulta el esquema para distinguir cantidad del movimiento de existencias finales. Conserva el motivo aportado por la usuaria y comprueba el saldo resultante.

Antes de editar un producto, lee su ficha y conserva configuración no mencionada. Consulta `service_inventory_requirements` para explicar el consumo por variante. Completar o cobrar citas ya descuenta sus consumibles: evita duplicarlo manualmente.

Si se solicita una propuesta de reposición, presenta cantidades y motivos basados en datos disponibles. El MCP no equivale a enviar un pedido al proveedor.
