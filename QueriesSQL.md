# Consultas SQL para Base de Datos Bancaria

## Enunciado 1: Clientes con múltiples cuentas y sus saldos totales

**Necesidad:** El banco necesita identificar a los clientes que tienen más de una cuenta, mostrar cuántas cuentas tiene cada uno y el saldo total acumulado en todas sus cuentas, ordenados por saldo total de mayor a menor.

**Consulta SQL:**
```sql
SELECT cli.id_cliente, cli.nombre, COUNT(cu.num_cuenta) AS cantidad_cuentas, SUM(cu.saldo) AS saldo_total FROM Cliente cli JOIN Cuenta cu ON cli.id_cliente = cu.id_cliente GROUP BY cli.id_cliente, cli.nombre HAVING COUNT(cu.num_cuenta) > 1 ORDER BY saldo_total DESC;
```

## Enunciado 2: Comparativa entre depósitos y retiros por cliente

**Necesidad:** El departamento de análisis financiero necesita comparar los montos totales de depósitos y retiros realizados por cada cliente, para identificar patrones de comportamiento financiero.

**Consulta SQL:**
```sql
SELECT cli.id_cliente, cli.nombre, COALESCE(SUM(CASE WHEN t.tipo_transaccion = 'deposito' THEN t.monto END), 0) AS total_depositos, COALESCE(SUM(CASE WHEN t.tipo_transaccion = 'retiro' THEN t.monto END), 0) AS total_retiros FROM Cliente cli JOIN Cuenta cue ON cli.id_cliente = cue.id_cliente JOIN Transaccion trans ON cue.num_cuenta = trans.num_cuenta GROUP BY cli.id_cliente, cli.nombre ORDER BY total_depositos DESC;
```

## Enunciado 3: Cuentas sin tarjetas asociadas

**Necesidad:** El departamento de tarjetas necesita identificar todas las cuentas que no tienen tarjetas asociadas para ofrecer productos específicos a estos clientes.

**Consulta SQL:**
```sql
SELECT cue.num_cuenta, cue.id_cliente, cli.nombre, cue.tipo_cuenta, cue.saldo, cue.fecha_apertura FROM Cuenta cue JOIN Cliente cli ON c.id_cliente = cli.id_cliente LEFT JOIN Tarjeta tarj ON c.num_cuenta = tarj.num_cuenta WHERE tarj.num_cuenta IS NULL;
```

## Enunciado 4: Análisis de saldos promedio por tipo de cuenta y comportamiento transaccional

**Necesidad:** La gerencia necesita un análisis comparativo del saldo promedio entre cuentas de ahorro y corriente, pero solo considerando aquellas cuentas que han tenido al menos una transacción en los últimos 30 días.

**Consulta SQL:**
```sql
SELECT cue.tipo_cuenta, ROUND(AVG(cue.saldo), 2) AS saldo_promedio FROM Cuenta cue JOIN Transaccion transa ON cue.num_cuenta = transa.num_cuenta WHERE transa.fecha >= CURRENT_DATE - INTERVAL '30 days' GROUP BY 
```

## Enunciado 5: Clientes con transferencias pero sin retiros en cajeros

**Necesidad:** El equipo de marketing digital necesita identificar a los clientes que utilizan transferencias pero no realizan retiros por cajeros automáticos, para dirigir campañas de banca digital.

**Consulta SQL:**
```sql
SELECT DISTINCT cli.id_cliente, cli.nombre FROM Cliente cli JOIN Cuenta cue ON cli.id_cliente = cue.id_cliente JOIN Transaccion transa ON cue.num_cuenta = transa.num_cuenta WHERE transa.tipo_transaccion = 'transferencia' AND cli.id_cliente NOT IN ( SELECT DISTINCT cli2.id_cliente FROM Cliente cli2 JOIN Cuenta cue2 ON cli2.id_cliente = cue2.id_cliente JOIN Transaccion trans2 ON cue2.num_cuenta = trans2.num_cuenta JOIN Retiro r ON trans2.id_transaccion = r.id_transaccion WHERE r.canal = 'cajero' );
```