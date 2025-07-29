# Consultas NoSQL para Sistema Bancario en MongoDB

A continuación se presentan 5 enunciados de consultas basados en las colecciones NoSQL del sistema bancario, junto con las soluciones utilizando operaciones avanzadas de MongoDB.

## 1. Análisis de Saldos por Tipo de Cuenta

**Enunciado:** El departamento financiero necesita un informe que muestre el saldo total, promedio, máximo y mínimo por cada tipo de cuenta (ahorro y corriente) para evaluar la distribución de fondos en el banco.

**Consulta MongoDB:**
```
db.cuentas.aggregate([
  { $unwind: "$cuentas" },
  {
    $group: {
      _id: "$cuentas.tipo_cuenta", // ahorro o corriente
      saldo_total: { $sum: "$cuentas.saldo" },
      saldo_promedio: { $avg: "$cuentas.saldo" },
      saldo_maximo: { $max: "$cuentas.saldo" },
      saldo_minimo: { $min: "$cuentas.saldo" },
      cantidad_cuentas: { $sum: 1 }
    }
  },
  {
    $project: {
      tipo_cuenta: "$_id",
      _id: 0,
      saldo_total: 1,
      saldo_promedio: 1,
      saldo_maximo: 1,
      saldo_minimo: 1,
      cantidad_cuentas: 1
    }
  }
])
```

## 2. Patrones de Transacciones por Cliente

**Enunciado:** El equipo de análisis de comportamiento necesita identificar los patrones de transacciones de cada cliente, mostrando la cantidad y el monto total de transacciones por tipo (depósito, retiro, transferencia) para cada cliente.

**Consulta MongoDB:**
```
db.transacciones.aggregate([
  {
    $group: {
      _id: {
        cliente: "$cliente_ref",
        tipo_transaccion: "$tipo_transaccion"
      },
      cantidad: { $sum: 1 },
      monto_total: { $sum: "$monto" }
    }
  },
  {
    $group: {
      _id: "$_id.cliente",
      transacciones: {
        $push: {
          tipo: "$_id.tipo_transaccion",
          cantidad: "$cantidad",
          monto_total: "$monto_total"
        }
      }
    }
  },
  {
    $lookup: {
      from: "clientes",
      localField: "_id",
      foreignField: "_id",
      as: "cliente"
    }
  },
  {
    $project: {
      _id: 0,
      cliente_id: "$_id",
      nombre: { $arrayElemAt: ["$cliente.nombre", 0] },
      cedula: { $arrayElemAt: ["$cliente.cedula", 0] },
      transacciones: 1
    }
  }
])
```

## 3. Clientes con Múltiples Tarjetas de Crédito

**Enunciado:** El departamento de riesgo crediticio necesita identificar a los clientes que poseen más de una tarjeta de crédito, mostrando sus datos personales, cantidad de tarjetas y el detalle de cada una.

**Consulta MongoDB:**
```
db.cuentas.aggregate([
  { $unwind: "$cuentas" },
  { $unwind: "$cuentas.tarjetas" },
  {
    $match: {
      "cuentas.tarjetas.tipo_tarjeta": "credito"
    }
  },
  {
    $group: {
      _id: "$_id",
      nombre: { $first: "$nombre" },
      cedula: { $first: "$cedula" },
      correo: { $first: "$correo" },
      direccion: { $first: "$direccion" },
      tarjetas_credito: { $push: "$cuentas.tarjetas" },
      cantidad_tarjetas_credito: { $sum: 1 }
    }
  },
  {
    $match: {
      cantidad_tarjetas_credito: { $gt: 1 }
    }
  },
  {
    $project: {
      _id: 0,
      nombre: 1,
      cedula: 1,
      correo: 1,
      direccion: 1,
      cantidad_tarjetas_credito: 1,
      tarjetas_credito: 1
    }
  }
])
```

## 4. Análisis de Medios de Pago más Utilizados

**Enunciado:** El departamento de marketing necesita conocer cuáles son los medios de pago más utilizados para depósitos, agrupados por mes, para orientar sus campañas promocionales.

**Consulta MongoDB:**
```
db.transacciones.aggregate([
  {
    $match: { tipo_transaccion: "deposito" }
  },
  {
    $addFields: {
      anio_mes: {
        $dateToString: { format: "%Y-%m", date: { $toDate: "$fecha" } }
      }
    }
  },
  {
    $group: {
      _id: {
        mes: "$anio_mes",
        medio_pago: "$detalles_deposito.medio_pago"
      },
      cantidad: { $sum: 1 },
      monto_total: { $sum: "$monto" }
    }
  },
  {
    $sort: {
      "_id.mes": 1,
      "cantidad": -1
    }
  },
  {
    $project: {
      _id: 0,
      mes: "$_id.mes",
      medio_pago: "$_id.medio_pago",
      cantidad: 1,
      monto_total: 1
    }
  }
])
```

## 5. Detección de Cuentas con Transacciones Sospechosas

**Enunciado:** El departamento de seguridad necesita identificar cuentas con patrones de transacciones sospechosas, definidas como aquellas que tienen más de 3 retiros en un mismo día con un monto total superior a 1,000,000 COP.

**Consulta MongoDB:**
```
db.transacciones.aggregate([
  {
    $match: {
      tipo_transaccion: "retiro"
    }
  },
  {
    $addFields: {
      fecha_dia: {
        $dateToString: { format: "%Y-%m-%d", date: { $toDate: "$fecha" } }
      }
    }
  },
  {
    $group: {
      _id: {
        num_cuenta: "$num_cuenta",
        fecha: "$fecha_dia"
      },
      total_retiros: { $sum: "$monto" },
      cantidad_retiros: { $sum: 1 },
      transacciones: { $push: "$$ROOT" }
    }
  },
  {
    $match: {
      cantidad_retiros: { $gt: 3 },
      total_retiros: { $gt: 1000000 }
    }
  },
  {
    $lookup: {
      from: "clientes",
      localField: "_id.num_cuenta",
      foreignField: "cuentas.num_cuenta",
      as: "cliente"
    }
  },
  {
    $project: {
      _id: 0,
      num_cuenta: "$_id.num_cuenta",
      fecha: "$_id.fecha",
      cantidad_retiros: 1,
      total_retiros: 1,
      transacciones: 1,
      cliente: {
        $arrayElemAt: ["$cliente", 0]
      }
    }
  }
])
```
