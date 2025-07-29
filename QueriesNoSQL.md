# Consultas NoSQL para Sistema Bancario en MongoDB

A continuación se presentan 5 enunciados de consultas basados en las colecciones NoSQL del sistema bancario, junto con las soluciones utilizando operaciones avanzadas de MongoDB.

## 1. Análisis de Saldos por Tipo de Cuenta

**Enunciado:** El departamento financiero necesita un informe que muestre el saldo total, promedio, máximo y mínimo por cada tipo de cuenta (ahorro y corriente) para evaluar la distribución de fondos en el banco.

**Consulta MongoDB:**
```javascript
db.clientes.aggregate([
  // Desenrolla las cuentas de cada cliente
  { $unwind: "$cuentas" },
  
  // Agrupa por tipo de cuenta y calcula los indicadores
  {
    $group: {
      _id: "$cuentas.tipo_cuenta",
      saldo_total: { $sum: "$cuentas.saldo" },
      saldo_promedio: { $avg: "$cuentas.saldo" },
      saldo_maximo: { $max: "$cuentas.saldo" },
      saldo_minimo: { $min: "$cuentas.saldo" },
      cantidad_cuentas: { $sum: 1 }
    }
  },
  
  // Proyecta los resultados con nombres legibles
  {
    $project: {
      _id: 0,
      tipo_cuenta: "$_id",
      saldo_total: 1,
      saldo_promedio: 1,
      saldo_maximo: 1,
      saldo_minimo: 1,
      cantidad_cuentas: 1
    }
  }
]);

```

## 2. Patrones de Transacciones por Cliente

**Enunciado:** El equipo de análisis de comportamiento necesita identificar los patrones de transacciones de cada cliente, mostrando la cantidad y el monto total de transacciones por tipo (depósito, retiro, transferencia) para cada cliente.

**Consulta MongoDB:**
```javascript
db.transacciones.aggregate([
  // Agrupar por cliente y tipo de transacción
  {
    $group: {
      _id: {
        cliente: "$cliente_ref",
        tipo_transaccion: "$tipo_transaccion"
      },
      total_transacciones: { $sum: 1 },
      monto_total: { $sum: "$monto" }
    }
  },

  // Reorganizar para agrupar por cliente
  {
    $group: {
      _id: "$_id.cliente",
      transacciones_por_tipo: {
        $push: {
          tipo_transaccion: "$_id.tipo_transaccion",
          total: "$total_transacciones",
          monto_total: "$monto_total"
        }
      }
    }
  },

  // Unir con la colección de clientes
  {
    $lookup: {
      from: "clientes",
      localField: "_id",
      foreignField: "_id",
      as: "cliente"
    }
  },
  { $unwind: "$cliente" },

  // Proyección final
  {
    $project: {
      _id: 0,
      cliente_id: "$_id",
      nombre: "$cliente.nombre",
      cedula: "$cliente.cedula",
      transacciones_por_tipo: 1
    }
  }
]);

```

## 3. Clientes con Múltiples Tarjetas de Crédito

**Enunciado:** El departamento de riesgo crediticio necesita identificar a los clientes que poseen más de una tarjeta de crédito, mostrando sus datos personales, cantidad de tarjetas y el detalle de cada una.

**Consulta MongoDB:**
```javascript
db.clientes.aggregate([
  // Desenrollamos las cuentas
  { $unwind: "$cuentas" },
  // Desenrollamos las tarjetas dentro de cada cuenta
  { $unwind: "$cuentas.tarjetas" },
  // Filtramos solo las tarjetas de crédito
  { $match: { "cuentas.tarjetas.tipo_tarjeta": "credito" } },
  // Agrupamos por cliente para contar tarjetas y recolectar sus datos
  {
    $group: {
      _id: "$_id",
      nombre: { $first: "$nombre" },
      cedula: { $first: "$cedula" },
      correo: { $first: "$correo" },
      direccion: { $first: "$direccion" },
      cantidad_tarjetas_credito: { $sum: 1 },
      tarjetas: {
        $push: {
          numero_tarjeta: "$cuentas.tarjetas.numero_tarjeta",
          fecha_emision: "$cuentas.tarjetas.fecha_emision",
          fecha_expiracion: "$cuentas.tarjetas.fecha_expiracion",
          cuenta_origen: "$cuentas.num_cuenta"
        }
      }
    }
  },
  // Nos quedamos solo con los clientes que tienen más de una tarjeta de crédito
  {
    $match: {
      cantidad_tarjetas_credito: { $gt: 1 }
    }
  },
  // Proyección final ordenada
  {
    $project: {
      _id: 0,
      nombre: 1,
      cedula: 1,
      correo: 1,
      direccion: 1,
      cantidad_tarjetas_credito: 1,
      tarjetas: 1
    }
  }
]);

```

## 4. Análisis de Medios de Pago más Utilizados

**Enunciado:** El departamento de marketing necesita conocer cuáles son los medios de pago más utilizados para depósitos, agrupados por mes, para orientar sus campañas promocionales.

**Consulta MongoDB:**
```javascript
db.transacciones.aggregate([
  // Filtramos solo los depósitos con medio de pago definido
  {
    $match: {
      tipo_transaccion: "deposito",
      "detalles_deposito.medio_pago": { $exists: true, $ne: null }
    }
  },
  // Extraemos el año y mes
  {
    $addFields: {
      mes: { $month: "$fecha" },
      anio: { $year: "$fecha" }
    }
  },
  // Agrupamos por año, mes y medio de pago
  {
    $group: {
      _id: {
        anio: "$anio",
        mes: "$mes",
        medio_pago: "$detalles_deposito.medio_pago"
      },
      cantidad: { $sum: 1 },
      monto_total: { $sum: "$monto" }
    }
  },
  // Ordenamos para visualización clara
  {
    $sort: {
      "_id.anio": 1,
      "_id.mes": 1,
      cantidad: -1
    }
  },
  // Proyección final legible
  {
    $project: {
      _id: 0,
      anio: "$_id.anio",
      mes: "$_id.mes",
      medio_pago: "$_id.medio_pago",
      cantidad: 1,
      monto_total: 1
    }
  }
]);

```

## 5. Detección de Cuentas con Transacciones Sospechosas

**Enunciado:** El departamento de seguridad necesita identificar cuentas con patrones de transacciones sospechosas, definidas como aquellas que tienen más de 3 retiros en un mismo día con un monto total superior a 1,000,000 COP.

**Consulta MongoDB:**
```javascript
db.transacciones.aggregate([
  // Filtrar solo retiros
  {
    $match: {
      tipo_transaccion: "retiro"
    }
  },
  // Crear campo con la fecha (solo año-mes-día, sin hora)
  {
    $addFields: {
      fecha_dia: {
        $dateToString: { format: "%Y-%m-%d", date: "$fecha" }
      }
    }
  },
  // Agrupar por cuenta y día
  {
    $group: {
      _id: {
        num_cuenta: "$num_cuenta",
        fecha_dia: "$fecha_dia"
      },
      cantidad_retiros: { $sum: 1 },
      monto_total: { $sum: "$monto" }
    }
  },
  // Filtrar casos sospechosos
  {
    $match: {
      cantidad_retiros: { $gt: 3 },
      monto_total: { $gt: 1000000 }
    }
  },
  // Enriquecer con información del cliente si se desea
  {
    $lookup: {
      from: "clientes",
      localField: "_id.num_cuenta",
      foreignField: "cuentas.num_cuenta",
      as: "cliente"
    }
  },
  { $unwind: "$cliente" },
  // Proyección final
  {
    $project: {
      _id: 0,
      num_cuenta: "$_id.num_cuenta",
      fecha: "$_id.fecha_dia",
      cantidad_retiros: 1,
      monto_total: 1,
      cliente: {
        nombre: "$cliente.nombre",
        cedula: "$cliente.cedula",
        correo: "$cliente.correo"
      }
    }
  }
]);


```