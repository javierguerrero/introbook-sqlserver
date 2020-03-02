
# MODULO 1: Trabajando con objetos de bases de datos

próximamente

<hr />

# MODULO 2: T-SQL y programabilidad de bases de datos

## Instrucciones DDL y DML

próximamente...

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

## Programabilidad  con T-SQL

Constructores básicos para la programación de base de datos en SQL Server
* Instrucciones básicas: condicionales, bucles, saltos
* Variables
* SQL dinámico
* Gestión de errores

### Instrucciones básicas

#### Consideraciones
* Cada instrucción sigue un patrón BEGIN...END para mantener un conjunto de lógica en un fragmento de código determinado.
* Podemos añadir el punto y coma (;) al final de las instrucciones
* Podemos terminar un lote utilizando la instrucción GO
    * Lote: un conjunto de lógica atómica
    * Todas las variables declaradas antes de GO se destruyen

#### Condiciones

```sql
IF DATENAME(weekday, GETDATE()) IN (N'Saturday', N'Sunday')
       SELECT 'Weekend';
ELSE 
       SELECT 'Weekday';
```

#### Bucles
```sql
DECLARE @Counter INT 
SET @Counter=1
WHILE (@Counter <= 10)
BEGIN
    PRINT 'The counter value is = ' + CONVERT(VARCHAR,@Counter)
    SET @Counter  = @Counter  + 1
END
```

```sql
DECLARE @Counter INT 
SET @Counter=1
WHILE (@Counter <= 10)
BEGIN
  PRINT 'The counter value is = ' + CONVERT(VARCHAR,@Counter)
  IF @Counter >=7
  BEGIN
	BREAK
  END
  SET @Counter  = @Counter  + 1
END
```

```sql
DECLARE @Counter INT 
SET @Counter=1
WHILE (@Counter <= 20)
BEGIN
	IF @Counter % 2 = 1
	BEGIN
		SET @Counter  = @Counter  + 1
		CONTINUE
	END
	PRINT 'The counter value is = ' + CONVERT(VARCHAR,@Counter)
	SET @Counter  = @Counter  + 1
END
```

```sql
--si usamos GO con un entero se crea un bucle
SELECT 1 AS number
GO 10
```

#### Saltos
Un salto consiste en viajar a otro punto del código designado con una etiqueta (label) 

```sql
DECLARE @Contador int;
SET @Contador = 1;
WHILE @Contador < 10
BEGIN
	SELECT @Contador
	IF @Contador = 4 GOTO Opcion_Uno --jumps to the first branch.
	IF @Contador = 5 GOTO Opcion_Dos --This will never execute.
	SET @Contador = @Contador + 1
END

Opcion_Uno:
	SELECT 'Eligio opción uno.'
	GOTO Salida; --Esto evitará que las opciones se ejecuten una despues de la otra

Opcion_Dos:
	SELECT 'Eligio opción dos.'
	GOTO Salida; --Esto evitará que las opciones se ejecuten una despues de la otra

Opcion_Tres:
	SELECT 'Eligio opción tres.'
	GOTO Salida; --Esto evitará que las opciones se ejecuten una despues de la otra

Salida:
	SELECT 'Adios'
```
### Variables

Se crean dentro de:
* un lote
* un procedimiento almacenado
* una función

Las variables de sistema (de solo lectura) son declaradas asi: @@+nombre_variable
* @@ROWCOUNT --> devuelve el número de filas afectadas por la última instrucción ejecutada

```sql
--declaración de variable
DECLARE @i;
DECLARE @s;

--establecer valores
SET @i = 1;
SET @s = 'Hello';

--otra forma de establecer valores
SELECT @i = 1;
SELECT @s = 'Hello';
```

### SQL dinámico

* Es una consulta que se construye concatenando partes de la instrucción.
* Podemos crear instrucciones dinámicamente (intercambiar nombres de tablas, crear filtros dinámicos, etc.).
* EXEC command executes a stored procedure or string passed to it.
* There is a possibility of SQL injection when you construct the SQL statement by concatenating strings from user input values.

```sql
DECLARE @SQL nvarchar(1000);
DECLARE @Pid varchar(50);
SET @Pid = '680';
SET @SQL = 'SELECT ProductID, Name, ProductNumber FROM Production.Product Where ProductID = '+ @Pid
EXEC (@SQL)
```

* El SQL dinámico es importante cuando tenemos que pasar consultas dinámicas desde la capa de aplicación.
* Formas de pasar una cadena de SQL a SQL Server
    * Crear la cadena en la capa de aplicación y pasársela a SQL Server
    * Usar consultas paramétrica (con marcadores de posición @variable)
    * Usar la API proporcionada por los lenguaje de programación (LINQ, por ejemplo)
    * Usar SPs con definición de SQL estático
    * Usar SPs con SQL dinámico
        * sp_ExecuteSQL

https://www.sqlshack.com/dynamic-sql-in-sql-server

### Gestión de errores

Formas de gestionar errores en T-SQL
* Examinar el valor de la variable de sistema @@ERROR
* Usar la instrucción TRY...CATCH

#### @@ERROR
Variable global que duelve 0 si la instrucción anterior no ha generado errores. Este valor cambia para cada intrucción de un lote, por lo que tenemos que guardarlo en una variable local si queremos conservar la información sobre el error

```sql
USE tempdb
GO

CREATE TABLE #FooData (id int NOT NULL);
DECLARE @value int; 

INSERT #FooData VALUES (@value)
IF @@ERROR <> 0
	PRINT 'Error inserting data!'
```

#### TRY...CATCH

```sql 
BEGIN TRY
	DECLARE @s smallint = CAST (11111111 as smallint);
END TRY
BEGIN CATCH
	SELECT 
		 ErrorNumber = ERROR_NUMBER()
		,ErrorMessage = ERROR_MESSAGE()
		,ErrorLine = ERROR_LINE()
		,ErrorSeverity = ERROR_SEVERITY()
END CATCH
```

Podemo usar la instrucción THROW para enviar la excepción al llamador. En las versiones previas a SQL Server 2012 sólo podíamos usar la función RAISEERROR.

```sql
THROW 55000, 'Error!', 1;
```
El parámetro `error_number` debe ser mayor a 50 000 que es el umbral por debajo del que SQL Server almacena mensajes de error para uso interno.

```sql
--Using THROW statement to rethrow an exception
CREATE TABLE #t1(
    id int primary key
);
GO

BEGIN TRY
    INSERT INTO #t1(id) VALUES(1);
    --cause error
    INSERT INTO #t1(id) VALUES(1);
END TRY
BEGIN CATCH
    PRINT('Raise the caught error again');
    THROW;
END CATCH
```
In this example, the first INSERT statement succeeded. However, the second one failed due to the primary key constraint. Therefore, the error was caught by the CATCH block was raised again by the THROW statement.

## Qué es un procedimiento almacenado

* Es un conjunto de instrucciones T-SQL
* Los SPs pueden estar anidados
* Tipos de SPs
    * Definidos por el usuario (creados por el developer)
    * Temporales (se crean en tempdb)
    * De sistema (devuelven información de metadatos)
    * Procedimientos CLR, heredados de un ensamblado CLR

### Trabajar con parámetros

Los valores que podemos pasar a un SP pueden ser: 
* parámetros de solo entrada
* parámetros de entrada y salida (palabra clave OUTPUT)
* parámetros con valores de tabla (table-valued parameters o TPV). Tablas de solo lectura (palabra clave READONLY)

Todo parámetro lleva un valor por defecto


Ejemplo de SP con parámetros de entrada y salida.
```sql
-- Creación del SP
USE [AdventureWorks2014]
GO

CREATE Procedure [dbo].[GetProductDetails]
	@ProductId		INT,
	@ProductNumber	VARCHAR(20)		OUTPUT,
	@ListPrice		MONEY			OUTPUT
AS
SELECT 
	@ProductNumber = ProductNumber,
	@ListPrice = ListPrice
FROM Production.Product 
WHERE ProductID = @ProductId

-- Uso del SP
USE [AdventureWorks2014]
GO

DECLARE	@return_value int,
		@ProductNumber varchar(20),
		@ListPrice money

EXEC	@return_value = [dbo].[GetProductDetails]
		@ProductId = 320,
		@ProductNumber = @ProductNumber OUTPUT,
		@ListPrice = @ListPrice OUTPUT

SELECT	@ProductNumber as N'@ProductNumber',
		@ListPrice as N'@ListPrice'

SELECT	'Return Value' = @return_value
GO
```


Ejemplo de parámetros con valores de tabla (table-valued parameters o TPV).
```sql
-- Create Department table
CREATE TABLE Department
(
    DepartmentID INT PRIMARY KEY,
    DepartmentName VARCHAR(30)
)

-- Create a TABLE TYPE and define the table structure
CREATE TYPE DeptType AS TABLE
(
    DeptId INT, 
    DeptName VARCHAR(30)
);

-- Declare a STORED PROCEDURE that has a parameter of table type
CREATE PROCEDURE InsertDepartment
    @InsertDept_TVP DeptType READONLY
AS
INSERT INTO Department(DepartmentID,DepartmentName)
SELECT * FROM @InsertDept_TVP;

-- Declare a table type variable and reference the table type
DECLARE @DepartmentTVP AS DeptType;

-- Using the INSERT statement and occupy the variable
INSERT INTO @DepartmentTVP(DeptId,DeptName)
VALUES (1,'Accounts'),
(2,'Purchase'),
(3,'Software'),
(4,'Stores'),
(5,'Maarketing');

-- We can now pass the variable to the procedure and Execute
EXEC InsertDepartment @DepartmentTVP;

-- Let’s see if the Data are inserted in the Department Table
SELECT * FROM Department
```

![](img/table-type.png)

Cada SP tiene un parámetro de valor de estado `@return_value`, que es el valor de estado que se devuelve automáticamente tras la ejecución del SP. Si no hay errores, `@return_value` es 0, de lo contrario es un valor entero correspondiente al error producido.

https://blog.cloudboost.io/how-to-use-sql-output-parameters-in-stored-procedures-578e7c4ff188
https://blog.sqlauthority.com/2008/08/31/sql-server-table-valued-parameters-in-sql-server-2008/

### Crear y modificar SPs

Las instrucciones DDL para crear y modificar procedimientos almacenados son:
* CREATE PROCEDURE
* ALTER PROCEDURE
Podemos mejorarlas con alguans palabras clave adicionales:
* WITH RECOMPILE: para forzar que se recree el plan de ejecución para cada ejecución
* WITH EXECUTE AS: para cambiar el contexto de seguridad de la ejecución
* WITH ENCRYPTION: para encriptar y ofuscar instrucciones de los procedimientos alamacenados


Ejemplos de como recompilar un SP en tiempo de ejecución
Fuente: https://blog.sqlauthority.com/2010/02/20/sql-server-recompile-stored-procedure-at-run-time/
```sql

-- Option 1: siempre recompilar
CREATE PROCEDURE dbo.PersonAge (@MinAge INT, @MaxAge INT)
WITH RECOMPILE
AS
SELECT *
FROM dbo.tblPerson
WHERE Age >= @MinAge AND Age <= @MaxAge
GO

-- Option 2: recompilar sola una vez
EXEC dbo.PersonAge 65,70 WITH RECOMPILE
```

Ejemplo de como cambiar el contexto de segeuridad la ejecución de un SP (impersonate)
Fuente: https://www.mssqltips.com/sqlservertip/1227/granting-permission-with-the-execute-as-command-in-sql-server/

```sql
-- 1. Crear SP
CREATE PROCEDURE dbo.usp_Demo2
WITH EXECUTE AS OWNER
AS
IF NOT EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].table_2') AND type in (N'U'))
CREATE TABLE table_2 (id int, data nchar(10))
INSERT INTO table_2
SELECT top 5 * from dbo.table_1;
GO

-- 2. Dar derechos de ejecución sobre el SP
GRANT EXEC ON dbo.usp_Demo1 TO test;
GO

-- 3. Ejecutar el SP
EXEC dbo.usp_Demo1
```

Ejemplo de como encriptar un SP
Fuente: https://www.c-sharpcorner.com/UploadFile/skumaar_mca/encrypt-the-stored-procedure-in-sql-server/

```sql
-- 1. Crear SP
CREATE PROCEDURE Proc_RetrieveProducts
WITH ENCRYPTION
AS
BEGIN
    SET NOCOUNT ON
    SELECT 
        ProductID, 
        ProductName, 
        ProductVendor 
   FROM Products
END

-- 2. Visualizar el SP en las tablas del sistemas (sobre la BD donde se creó el SP)
select sc.id, so.name, sc.ctext, sc.text
from SYScomments sc 
inner join sys.objects so on so.object_id = sc.id

-- 3. Tratar de visualizar esquema de SP encriptado 
SP_HELPTEXT Proc_RetrieveProducts
```

> Once stored procedure is created then the stored procedure should be manually stored some where in the local system and it should be used for the changes.

### Opciones de los SPs

NOCOUNT 
* Si está en ON no se devolverá al llamador ningún mensaje de estado de las filas afectadas
* A pesar que NOCOUNT esté establecida en ON, la variable de sistema @@ROWCOUNT continúa devolviendo el número correcto de las filas afectadas

Ejemplo de uso de NOCOUNT
Fuente: https://www.sqlshack.com/set-nocount-on-statement-usage-and-performance-benefits-in-sql-server/

```sql
-- 1. Crear tabla
USE CourseDatabase;
GO
CREATE TABLE tblEmployeeDemo
(
    Id      INT
    PRIMARY KEY, 
    EmpName NVARCHAR(50), 
    Gender  NVARCHAR(10),
);

-- 2. insertar registros
USE CourseDatabase
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (1, N'Grace', N'Female')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (2, N'Gordon', N'Male')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (3, N'Jaime', N'Female')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (4, N'Ruben', N'Female')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (5, N'Makayla', N'Male')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (6, N'Barry', N'Female')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (7, N'Ramon', N'Male')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (8, N'Douglas', N'Male')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (9, N'Julian', N'Female')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (10, N'Sarah', N'Female')
GO

-- 3. seleccionar registros
SELECT * FROM tblEmployeeDemo;

-- 4. Crear SP
CREATE PROCEDURE SP_tblEmployeeDemo
AS
BEGIN
    SELECT *
    FROM tblEmployeeDemo;
END;

-- 5. Ejecutar SP
Exec SP_tblEmployeeDemo


-- 6. Execute the insert statement after enabling the NOCOUNT 
TRUNCATE TABLE tblEmployeeDemo;
GO

SET NOCOUNT ON

USE CourseDatabase
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (1, N'Grace', N'Female')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (2, N'Gordon', N'Male')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (3, N'Jaime', N'Female')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (4, N'Ruben', N'Female')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (5, N'Makayla', N'Male')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (6, N'Barry', N'Female')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (7, N'Ramon', N'Male')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (8, N'Douglas', N'Male')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (9, N'Julian', N'Female')
GO
INSERT [dbo].[tblEmployeeDemo] ([Id], [EmpName], [Gender]) VALUES (10, N'Sarah', N'Female')
GO

-- 7. Execute the Select statement with NOCOUNT ON
SET NOCOUNT ON
SELECT * FROM tblEmployeeDemo;

-- 8. Alter the procedure and add the SET NOCOUNT statement 
ALTER PROCEDURE SP_tblEmployeeDemo
AS
BEGIN
    SET NOCOUNT ON;
    SELECT * FROM tblEmployeeDemo;
END;

-- 9. 
Exec SP_tblEmployeeDemo
```

WITH RESULT SETS 
* SQL Server 2012+
* Definir los metadatos del conjunto de resultados

### Ventajas y desventajas de los SPs

Ventajas
* Reducción del tráfico de red
* Seguridad
    * Evita ataques de inyección de SQL
    * Sólo dar permiso a los PS que queremos que se ejecuten
    * Evitar acceso directo a los objetos de BDs
* Modularidad y facilidad de mantenimiento
    * Reutilización de código
* Rendimiento mejorado
    * Los SPs se compilan justo en el momento de su creación
    * Cuando un SP se compila se crea el plan de ejecución óptimo y queda en memoria hasta que se lo modifique.

Desventajas
* La estricta vinculación con el RDBMS
    * Quedamos pegamos al lenguaje y motor de BD.
* La vinculación entre la capa de aplicación y el conjunto de resultados
    * Acoplamiento. La capa de aplicación tiene que conocer las especificaciones de salida de un módulo de SQL.
* El uso de la lógica de negocio en los procedimientos almacenados
    * Demasiada lógica de negocio en SPs crean "cuellos de botella"


## Funciones personalizadas

Una función SIEMPRE devuelve un resultado

### Tipos de funciones
http://microsoftsqlsecret.fullblog.com.ar/funciones-sql-server-funciones-escalares-y-funciones-con-valores-de-t.html

Función escalar
```sql
-- Creamos la funcion con el nombre IVA
-- Indicamos el parámetro de entrada y tipo: @cantidad money
Create function IVA (@cantidad money)
Returns money -- Indicamos el tipo de parámetros que retornará la función.
as
 -- Encapsulamos el conjunto de funciones dentro de un Begin y un end.
Begin
	-- Dentro de la sentencia o funcion podemos manejar variables
	--aunque puede no ser necesario para ser más eficiente la función.
	-- por cuestiones de ejemplo usamos la variable @resultado
	Declare @resultado money
	set @resultado  = @cantidad * 0.16
	-- Cuando terminamos la función ponemos el Return
	Return (@resultado) -- Y la variable que devolverá la función @resultado.
end

-- Llamar a la función
Select campo_producto, campo_unidadprecio, dbo.iva(campo_unidadprecio) as iva from tabla
```

Funciones con valores de tabla

```sql
--Creamos una función que se le ingrese por parámetro el país y devuelva los clientes de ese país:
Create function ListadoPais (@pais varchar(100))
returns @clientes table -- Decimos que retornamos como resultado una variable de tipo tabla y la declaramos
	(
	customerid varchar(5), 
	companyname varchar(50), -- Se definen las columnas que se necesiten
	contactname varchar(100), 
	country varchar(100)
	)
as
-- Iniciamos la función. La variable tipo tabla @clientes es la que vamos a devolver como resultado de la función
begin
	-- Insertamos a la variable tipo tabla @clientes:
	Insert @clientes 
	select customerid, companyname, contactname, country 
	from customers 
	where country = @pais -- variable @pais de entrada en la función
	Return
end

-- llamar a la función
Select * from dbo.ListadoPais('Argentina')
```

Funciones con valores de tablas en línea
```sql
Create funcion ListadoPais2 (@pais varchar(100))
returns table
as
return
(
	select customerid, companyname,contactname, country 
	from customers 
	where country = @pais
)
```

### Limitaciones
* No podemos cambiar el estado de la BD
* TRY...CATCH no está permitido
* ORDER BY no garantiza la ordenación del conjunto de resultados

### Funciones integradas

https://www.sqlshack.com/es/como-utilizar-las-funciones-integradas-de-sql-server-y-crear-funciones-escalares-definidas-por-el-usuario/

```sql
SELECT CHOOSE(2,'Gold','Silver','Bronze')
```
https://www.essentialsql.com/how-to-use-the-choose-function-with-select/


# MODULO 3: Trabajando con índices

## Acerca de los índices
* Un índice es una estructura de datos que permite un acceso mucho más rápido a los datos.
* Los índices nos ayudan a reducir las operaciones de entrada y salida y el consumo de recursos del sistema.
* Si en una tabla creamos demasiados índices, se reducirá el rendimiento al escribir en ella.
* Vista indizaad: Usar índices en vistas que empleen muchos agregados, combinaciones de tablas o ambos operadores. 

```
Con SQL Server, algunos índices se crean automáticamente. Cuando
intentamos aplicar una clave principal o una restricción única, se crea
automáticamente un índice único.
```

## Tipos de índices
https://www.guru99.com/clustered-vs-non-clustered-index.html

### Clustered index
* With a clustered index the rows are stored physically on the disk in the same order as the index. Therefore, there can be only one clustered index.
* It is generally faster to read from a clustered index if you want to get back all the columns. You do not have to go first to the index and then to the table.
* Writing to a table with a clustered index can be slower, if there is a need to rearrange the data.

![](img/clustered-index.png)

Crear un clustered index
```sql
--Indice compuesto
CREATE CLUSTERED INDEX IX_tblStudent_Gender_Score
ON student(gender ASC, total_score DESC)
```

### Non-clustered index
* With a non clustered index there is a second list that has pointers to the physical rows. You can have many non clustered indices, although each new index will increase the time it takes to write new records. 

Crear un non-clustered index
```sql
CREATE NONCLUSTERED INDEX IX_tblStudent_Name
ON student(name ASC)
```
![](img/non-clustered-index.png)

### Tips

recuperar todos los índices de la tabla
```sql
EXECUTE sp_helpindex student
```

Links
* http://skatageri.blogspot.com/p/index.html
* https://www.sqlshack.com/es/cual-es-la-diferencia-entre-indices-agrupados-y-no-agrupados-en-sql-server/