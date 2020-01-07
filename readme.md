# SQL

## Consultas y subconsultas

### ¿Qué es una conssulta?

* A query is a question or inquiry to a set of data. We use SQL, or Structured Query Language, to retrieve data from databases.

### ¿Cómo funciona una consulta?

* La lógica que sigue el motor para el procesado de consultas es la siguiente:

![](img/orden-procesamiento-consulta.png)

![](img/pasos-procesamiento-consulta.png)

* Cada paso genera una **tabla virtual** que se usa para el paso siguiente.

### ¿Qué es una subconsulta?

* Es una consulta anidada que es parte de otra instrucción SELECT, INSERT, UPDATE, DELETE o bien otra subconsulta.

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

*  Lugares donde se pueden usar subconsultas
    * En un filtro IN
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
    * En instrucciones UPDATE, INSERT y DELETE
    * En filtros con operadores de comparación (mayor que, menor que, igual, etc.)
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
    * Con la función NOT EXISTS | EXISTS
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
