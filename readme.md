
# MODULO 1: Trabajando con objetos de bases de datos

próximamente

# MODULO 2: T-SQL y programabilidad de bases de datos

## Instrucciones DML

próximamente

## Consultas y subconsultas

### ¿Qué es una consulta?

A query is a question or inquiry to a set of data. We use SQL, or Structured Query Language, to retrieve data from databases.

### ¿Cómo funciona una consulta?

La lógica que sigue el motor para el procesado de consultas es la siguiente:

![](img/orden-procesamiento-consulta.png)

![](img/pasos-procesamiento-consulta.png)

Cada paso genera una **tabla virtual** que se usa para el paso siguiente.

### ¿Qué es una subconsulta?

Es una consulta anidada que es parte de otra instrucción SELECT, INSERT, UPDATE, DELETE o bien otra subconsulta.
```sql
SELECT
    T.CustomerID,
    T.MaxTotalQuantity
FROM 
    (
        SELECT
            CustomerID,
            MaxTotalQuantity = MAX(TotalQuantity)
        FROM Sales.SalesOrders
        WHERE TotalQuantity > 0
        GROUP BY CustomerID
    ) AS T
WHERE T.CustomerID = 10
```

#### Lugares donde se pueden usar subconsultas

En un filtro IN
```sql
SELECT 
    SalesOrderID,
    SalesOrderDate,
    TotalQuantity,
    ExpiredDate,
    CustomerID
FROM
    Sales.SalesOrders SO
WHERE
    CustomerID IN (SELECT CustomerID FROM Sales.Customers C WHERE CustomerID > 1);
```

En instrucciones UPDATE, INSERT y DELETE
```sql
```

En filtros con operadores de comparación (mayor que, menor que, igual, etc.)
```sql
SELECT 
    SalesOrderID,
    SalesOrderDate,
    TotalQuantity,
    ExpiredDate,
    CustomerID
FROM
    Sales.SalesOrders SO
WHERE 
    TotalQuantity > (SELECT MAX(TotalQuantity) FROM Sales.SalesOrders);
```

Con la función NOT EXISTS | EXISTS
```sql
SELECT 
    SalesOrderID,
    SalesOrderDate,
    TotalQuantity,
    ExpiredDate,
    CustomerID
FROM
    Sales.SalesOrders SO
WHERE 
    NOT EXISTS (SELECT CustomerID FROM Sales.Customers C WHERE FirstName LIKE 'A%');
```

En lugar de una expresión (por ejemplo, en una instrucción SELECT)
```sql
SELECT 
    SalesOrderID,
    TotalQuantity,
    ExpiredDate,
    CustomerID,
    Days = (SELECT DATEDIFF(DAY, GETDATE(), ExpiredDate)) FROM Sales.SalesOrders WHERE SalesOrderID = 2)
FROM 
    Sales.SalesOrders SO
```


#### Rendimiento en subconsultas

Como cada FROM+JOIN crea un producto cartesiano entre tablas, el uso de subconsultas es una buena práctica para mejorar el rendimiento de la ejecución.

```
Caso práctico
Tenemos:
    tabla maestra --> Sales.SalesOrders
    tabla de detalles --> Sales.SalesOrderDetails

Se pide:
leer un cierto número de pedidos con sus detalles para un intervalo de tiempo

Soluciones:
    una instrucciòn SELECT con un JOIN para enlazar las tablas utilizando la columna SalesOrderId

    uso de subconsultas para filtrar registros maestros antes de unirlos con las otras tablas de detalle
```

### Common Table Expressions (CTE)

* Ventajas
    * modularidad
    * facilidad de mantenimiento
    * pueden definirse en rutinas
* Acerca de la sintaxis
    * Se debe usar la palabra clave WITH para declarar la CTE
    * Un WITH no puede tener otra definición WITH anidada
![](img/cte_syntax.png)


* Links:
    * https://www.campusmvp.es/recursos/post/SQL-Server-Expresiones-de-tabla-comunes.aspx


#### Ejemplo de una CTE NO recursiva

```sql
WITH CustomersCTE (CustomerName, Quantity, ID)
AS
(
    SELECT DISTINCT
        C.FirstName + ' ' + C.LastName,
        SO.TotalQuantity,
        C.CustomerID
    FROM
        Sales.Customers C
    JOIN 
        Sales.SalesOrders SO ON C.CustomerID = SO.CustomerID
)

SELECT 
    CustomerName,
    Quantity,
    ID
FROM
    CustomersCTE
```

#### Ejemplo de CTE recursiva

* Útil para navegar a través de jerarquías 
* Nivel máximo de recursiones: 32767

Ejemplo: mostrar un jefe y sus subordinados

```sql
WITH CTE_Employees AS
(
    SELECT 
        EmployeeID, 
        ManagerID, 
        FirstName, 
        LastName, 
        1 as Pos
    FROM HR.Employees 
    WHERE EmployeeID = 10

    UNION ALL

    SELECT 
        E.EmployeeID, 
        E.ManagerID, 
        E.FirstName, 
        E.LastName, 
        (C.Pos + 1) as Pos
    FROM HR.Employees E
    INNER JOIN CTE_Employees C ON E.ManagerID = C.EmployeeID
)

SELECT * FROM CTE_Employees
```

## Extendiendo conjunto de resultados

Hay muchas maneras de extender resultados:

* Añadir columnas calculadas
* Enlazar tablas con operadores JOIN o APPLY
* Combinar resultados con operador UNION 

### Añadir columnas calculadas

Las columnas calculadas constituyen el resultado de una expresión. Las columnas calculadas se pueden usar en:
* la lista de SELECT
* cláusula WHERE
* cláusula ORDER BY

Ejemplo que muestra como añadir una expresión a la lista de SELECT para extender el conjunto de columnas:

```sql
SELECT 
    SOD.SalesOrderID,
    SOD.OrderNumber,
    SOD.Quantity,
    SOD.Price,
    ItemTotalPrice = SOD.Quantity * SOD.Price
FROM
    Sales.SalesOrderDetails SOD
```

> **Performance tip:** Normalmente, una columna calculada es un campo virtual que no persiste en la base de datos. Sin embargo, una razón para persistir una **columna calculada** es si tenemos que aplicarle un filtro (WHERE), también con el fin de permitir al diseñador crear un índice para acelerar las búsquedas.

Formas de extender el cojunto de columnas:
* columnas calculadas condicionales
* usando funciones de categoría: 
    * RANK()
    * ROW_NUMBER()
    * DENSE_RANK()
    * NTILE(n)
* usando funciones analíticas
    * CUME_DIST()
    * LAG()
    * LEAD()
    * PERCENTILE_CONT()
    * PERCENTILE_DISC()
    * PERCENTILE_RANK()
    * FIRST_VALUE()
    * LAST_VALUE()


### Enlazar tablas con operadores JOIN o APPLY

* 

### Combinar resultados con operador UNION 




# MODULO 3: Trabajando con índices

próximamente

# MODULO 4: Trabajando con transacciones

próximamente

# MODULO 5: Seguridad de SQL Server

próximamente

# MODULO 6: Supervisión y solución de problemas

próximamente