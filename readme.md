
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
* Relacionar tablas: operadores JOIN o APPLY
* Añadir filas: operador UNION 

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
* usando funciones de categoría: funciones útiles para enumerar registros, pero también para examinar la calidad de datos.
    * RANK()
    * ROW_NUMBER()
        * http://bit.ly/2TkxfRP
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

Identificar los registros dobles
```sql
SELECT 
	IDMultiValue, 
	SampleValue, 
	SampleDescription,
	repeat_number = ROW_NUMBER() OVER (PARTITION BY IDMultiValue ORDER BY IDMultiValue)
FROM dbo.MultiValues
```

Cómo usar una CTE para eliminar registros duplicados
```sql
--recopila los valores repetidos
WITH MultiCTE AS
(
	SELECT 
		IDMultiValue,
		SampleValue,
		SampleDescription,
		repeat_number = ROW_NUMBER() OVER (PARTITION BY IDMultiValue ORDER BY IDMultiValue)
	FROM 
		dbo.MultiValues
)

--borrar la instrucción con la CTE
DELETE FROM MultiCTE WHERE repeat_number > 1
```

> **Nota:** Una expresión es determinista si esta devuelve siempre el mismo resultado para un conjunto de entrada. La función GETDATE() no es determinista.


### Relacionar tablas: operadores JOIN o APPLY

* El operador JOIN nos permite hacer combinaciones lógicas entre tablas
* Podemos hacer la operación JOIN con más de 2 tablas
* Normalmente, las relaciones se establecen entre la clave principal de una tabla y la clave externa correspondiente
* Each join type specifies how SQL Server uses data from one table to select rows in another table.
* Tipos de JOIN
    * INNER JOIN
        * También se puede usar la palabra JOIN
    * OUTER JOIN
        * LEFT OUTER JOIN = LEFT JOIN
        * RIGHT OUTER JOIN = RIGTH JOIN
        * FULL OUTER JOIN = FULL JOIN
    * CROSS JOIN
        * Productos cartesianos
        * No necesita ninguna cláusula ON ya que no hay ninguna clave de coincidencia
* Operador APPLY
    * to perform join operations between a physical table and table valued function
    * permite invocar por cada fila de una consulta una **función que devuelva un tipo tabla**. En otras palabras, hace una proyección de la expresión izquierda a través de la función (o expresión de tipo tabla) que se le indica.
    * Tipos
        * CROSS APPLY: parecido a INNER JOIN
        * OUTER APPLY: parecido a LEFT JOIN
    * https://www.sqlshack.com/the-difference-between-cross-apply-and-outer-apply-in-sql-server/

```sql
-- Crear tabla A (tabla Izquierda)
CREATE TABLE A
(
a INT
);

-- Crear tabla B (tabla derecha)
CREATE TABLE B
(
b INT
);

-- Insertar datos
Insert into A (a) Values (1);
Insert into A (a) Values (2);
Insert into A (a) Values (3);
Insert into A (a) Values (4);
Insert into B (b) Values (3);
Insert into B (b) Values (4);
Insert into B (b) Values (5);
Insert into B (b) Values (6);
GO

-- Tabla A
SELECT * FROM A;

-- Tabla B
SELECT * FROM B;

/* Inner Join. */
-- Unión interna, filas que ambas tablas tienen en común.
select * from A INNER JOIN B on A.a = B.b;

/* Left outer join */
-- Unión externa por la izquierda, todas las filas de A (tabla izquierda) relacionadas con B, así estas tengan o no coincidencias.
select * from A LEFT OUTER JOIN B on A.a = B.b;

/* Right outer join */
-- Unión externa por la derecha, todas las filas de B (tabla derecha), así estas tengan o no coincidencias con A.
select * from A RIGHT OUTER JOIN B on A.a = B.b;

/* Full outer join */
-- Unión externa completa, unión externa por la izquierda unida a unión externa por la derecha. 
select * from A FULL OUTER JOIN B on A.a = B.b;

/* Cross join*/
select * from A CROSS JOIN B
```

Links:
* https://www.sqlservertutorial.net/sql-server-basics/sql-server-joins/


### Añadir filas: operador UNION 

* El número de columnas no cambia
* Reglas para usar el operador UNION
    * El número y orden de las columnas de todas las consultas deben ser el mismo
    * Los tipos de dato de cada columna deben ser compatibles con los del conjunto de datos añadido
* Es posible que haya duplicados en las filas
    * UNION --> elimina duplicados
    * UNION ALL --> para mantener filas repetidas
* Se puede usar ORDER BY al final de los conjuntos de datos combinados

```sql
USE tempdb;
GO
-- primer conjunto
CREATE TABLE #FirstSet (id int, val varchar(10));

-- segundo conjunto
CREATE TABLE #SecondSet (id int, val varchar(10), number smallint);

-- tercer conjunto
CREATE TABLE #ThirdSet (id int, number smallint);

-- INSERT VALUES
-- ///////////////////////////////////////////
INSERT INTO #FirstSet ( id, val )
VALUES
( 1, 'ONE')
, ( 2, 'TWO')
, ( 3, 'THREE');

INSERT INTO #SecondSet ( id, val, number )
VALUES
( 3, 'THREE', 300 )
, ( 4, 'FOUR', 400 )
, ( 5, 'FIVE', 500 );

INSERT INTO #ThirdSet ( id, number )
VALUES
( 2, 200 )
, ( 3, 300 )
, ( 4, 500 );
-- ///////////////////////////////////////////

-- solo UNION (sin duplicados)
SELECT id, val FROM #FirstSet

UNION

SELECT id, val FROM #SecondSet

UNION

SELECT id, val = CASE
					WHEN id = 2 THEN 'TWO'
					WHEN id = 3 THEN 'THREE'
					WHEN id = 4 THEN 'FOUR'
				END
FROM #ThirdSet;

DROP TABLE #FirstSet;
DROP TABLE #SecondSet;
DROP TABLE #ThirdSet;
GO
```

## Objetos temporales

* Base de datos para objetos temporales: tempdb
* Se recomienda que tempdb sea una BD separada físicamente en RAID
*  Creación de objetos temporales
    * '#' --> tabla temporal local
    * '@' --> variable de tabla

### Tablas temporales locales

* visibles para su creador durante una misma conexión
* Se borran automáticamente una vez que el creador se desconecta de la instancia

```sql
USE tempdb;
GO

CREATE TABLE #FirstSet (id int, val varchar(10));
GO

DROP TABLE #FirstSet;
GO
```

### Tablas temporales globales

* Visible para cualquier usuario
* Se borrarn cuando todos los usuarios que están referenciándola se desconectan de la instancia
* Ojo con el DROP TABLE porque estas tablas son de ámbito global (evitar usarlas)

```sql
USE tempdb;
GO

CREATE TABLE ##FirstSet (id int, val varchar(10));
GO    
```

### Variables de tabla
* Cuando hay poca memoria libre se almacenan en tempdb, sino en memoria
* Podemos:
    * En SP: pasarle datos en forma de variable de tabla
    * En funciones: devolver variable tabla

```sql
DECLARE <table_name> TABLE (<column_definition>);
```

### Ventajas/desventajas de usar tablas temporales

* Ventajas
    * la división de los datos paso a paso para mejorar las consultas grandes (optimización)
    * los objetos se autodestruyen
    * ...

* Desventajas
    * se crea una tabla temporal local para cada usuario
    * pueden crear un cuello de botella en la entrada y salida dentro tempdb


Ejemplo de como dividir una consulta en varias tablas. Vamos a intentar minizar los productos cartesianos

```sql

--consulta original
select 
	H.DueDate,
	H.Comment,
	A.AddressLine1,
	A.AddressLine2,
	A.City,
	PR.FirstName,
	PR.LastName,
	D.UnitPrice,
	D.UnitPriceDiscount,
	SUM(D.UnitPrice) as TotalPrice
from Sales.SalesOrderHeader H
	join Sales.SalesOrderDetail D on H.SalesOrderID = D.SalesOrderID
	join Production.Product P on D.ProductID = P.ProductID
	left join Sales.Customer C on H.CustomerID = C.CustomerID
	left join Person.Person PR on C.PersonID = PR.BusinessEntityID
	left join Person.Address A on A.AddressID = H.BillToAddressID
where 
	H.ModifiedDate >= '20010101'
	and P.SellEndDate >= '20010101'
	and PR.LastName like 'a%'
group by
	H.DueDate,
	H.Comment,
	A.AddressLine1,
	A.AddressLine2,
	A.City,
	PR.FirstName,
	PR.LastName,
	D.UnitPrice,
	D.UnitPriceDiscount

--consulta optimizada
declare @TempOrders table ( -- declaro una variable de tipo tabla
	DueDate datetime,
	Comment nvarchar(128),
	UnitPrice money, 
	UnitPriceDiscount money,
	ProductID int, 
	CustomerID int, 
	BillToAddressID int
);

select 
	H.DueDate,
	H.Comment,
	D.UnitPrice,
	D.UnitPriceDiscount,
	D.ProductID,
	H.CustomerID,
	H.BillToAddressID
from Sales.SalesOrderHeader H
	join Sales.SalesOrderDetail D on H.SalesOrderID = D.SalesOrderID
where H.ModifiedDate >= '20010101'


select 
	TMP.DueDate,
	TMP.Comment,
	A.AddressLine1,
	A.AddressLine2,
	A.City,
	PR.FirstName,
	PR.LastName,
	TMP.UnitPrice,
	TMP.UnitPriceDiscount,
	SUM(TMP.UnitPrice) as TotalPrice
from @TempOrders TMP
	join Production.Product P on TMP.ProductID = P.ProductID
	left join Sales.Customer C on TMP.CustomerID = C.CustomerID
	left join Person.Person PR on C.PersonID = PR.BusinessEntityID
	left join Person.Address A on A.AddressID = TMP.BillToAddressID
where 
	P.SellEndDate >= '20010101'
	and PR.LastName like 'a%'
group by
	TMP.DueDate,
	TMP.Comment,
	A.AddressLine1,
	A.AddressLine2,
	A.City,
	PR.FirstName,
	PR.LastName,
	TMP.UnitPrice,
	TMP.UnitPriceDiscount

```




# MODULO 3: Trabajando con índices

próximamente

# MODULO 4: Trabajando con transacciones

próximamente

# MODULO 5: Seguridad de SQL Server

próximamente

# MODULO 6: Supervisión y solución de problemas

próximamente