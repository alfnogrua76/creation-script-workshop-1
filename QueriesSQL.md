# Consultas SQL para Base de Datos Bancaria

## Enunciado 1: Clientes con múltiples cuentas y sus saldos totales

**Necesidad:** El banco necesita identificar a los clientes que tienen más de una cuenta, mostrar cuántas cuentas tiene cada uno y el saldo total acumulado en todas sus cuentas, ordenados por saldo total de mayor a menor.

**Consulta SQL:**
```
SELECT 
    c.id_cliente,
    c.nombre,
    COUNT(ct.num_cuenta) AS cantidad_cuentas,
    SUM(ct.saldo) AS saldo_total
FROM 
    Cliente c
JOIN 
    Cuenta ct ON c.id_cliente = ct.id_cliente
GROUP BY 
    c.id_cliente, c.nombre
HAVING 
    COUNT(ct.num_cuenta) > 1
ORDER BY 
    saldo_total DESC;


```

## Enunciado 2: Comparativa entre depósitos y retiros por cliente

**Necesidad:** El departamento de análisis financiero necesita comparar los montos totales de depósitos y retiros realizados por cada cliente, para identificar patrones de comportamiento financiero.

**Consulta SQL:**
```
SELECT 
    c.id_cliente,
    c.nombre,
    COALESCE(SUM(CASE WHEN t.tipo_transaccion = 'deposito' THEN t.monto END), 0) AS total_depositos,
    COALESCE(SUM(CASE WHEN t.tipo_transaccion = 'retiro' THEN t.monto END), 0) AS total_retiros,
    COALESCE(SUM(CASE WHEN t.tipo_transaccion = 'deposito' THEN t.monto END), 0) -
    COALESCE(SUM(CASE WHEN t.tipo_transaccion = 'retiro' THEN t.monto END), 0) AS balance
FROM Cliente c
JOIN Cuenta cu ON c.id_cliente = cu.id_cliente
JOIN Transaccion t ON cu.num_cuenta = t.num_cuenta
GROUP BY c.id_cliente, c.nombre
ORDER BY balance DESC;
```

## Enunciado 3: Cuentas sin tarjetas asociadas

**Necesidad:** El departamento de tarjetas necesita identificar todas las cuentas que no tienen tarjetas asociadas para ofrecer productos específicos a estos clientes.

**Consulta SQL:**
```
SELECT 
    c.id_cliente,
    c.nombre,
    cu.num_cuenta,
    cu.tipo_cuenta,
    cu.saldo,
    cu.fecha_apertura
FROM Cliente c
JOIN Cuenta cu ON c.id_cliente = cu.id_cliente
LEFT JOIN Tarjeta t ON cu.num_cuenta = t.num_cuenta
WHERE t.num_cuenta IS NULL
ORDER BY cu.fecha_apertura DESC;

```

## Enunciado 4: Análisis de saldos promedio por tipo de cuenta y comportamiento transaccional

**Necesidad:** La gerencia necesita un análisis comparativo del saldo promedio entre cuentas de ahorro y corriente, pero solo considerando aquellas cuentas que han tenido al menos una transacción en los últimos 30 días.

**Consulta SQL:**
```
SELECT 
    cu.tipo_cuenta,
    ROUND(AVG(cu.saldo), 2) AS saldo_promedio
FROM Cuenta cu
WHERE cu.num_cuenta IN (
    SELECT DISTINCT t.num_cuenta
    FROM Transaccion t
    WHERE t.fecha >= CURRENT_DATE - INTERVAL '30 days'
)
GROUP BY cu.tipo_cuenta;

```

## Enunciado 5: Clientes con transferencias pero sin retiros en cajeros

**Necesidad:** El equipo de marketing digital necesita identificar a los clientes que utilizan transferencias pero no realizan retiros por cajeros automáticos, para dirigir campañas de banca digital.

**Consulta SQL:**
```
SELECT DISTINCT c.id_cliente, c.nombre, c.correo
FROM Cliente c
JOIN Cuenta ct ON c.id_cliente = ct.id_cliente
JOIN Transaccion t ON ct.num_cuenta = t.num_cuenta
JOIN Transferencia trf ON t.id_transaccion = trf.id_transaccion
WHERE c.id_cliente NOT IN (
    SELECT DISTINCT c2.id_cliente
    FROM Cliente c2
    JOIN Cuenta ct2 ON c2.id_cliente = ct2.id_cliente
    JOIN Transaccion t2 ON ct2.num_cuenta = t2.num_cuenta
    JOIN Retiro r ON t2.id_transaccion = r.id_transaccion
    WHERE r.canal ILIKE 'cajero'
);

```
