# DAX Useful Functions
Repository for dax coding.

---

### Cálculos comunes

**Denominador porcentaje**

ALL - ALL EXCEPT Returns all the rows in a table, or all the values in a column, ignoring any filters that might have been applied.

```
Example:
CALCULATE ( 
	SUM ( Table[Column] ) ;
	ALLEXCEPT ( Table ; Table[FilterColumn1] ; ... ; Table[FilterColumnZ] )
)
```

**Cálculo de Sumarización**

SUMMARIZE Creates a summary the input table grouped by the specified columns.
ADDCOLUMNS Returns a table with new columns specified by the DAX expressions.
Summarize has failures with the addition of the aggregation calculation but it is excelent for getting the distinct table of certain columns.
```
Example:
ADDCOLUMNS (
	SUMMARIZE ( Table, Table[Column] ),
	"@VirtualColumn", COUNTROWS ( Table ) 
)
```

**Calculate sin pisar contexto**

Recordemos que CALCULATE superpone sus filtros sobre el contexto de filtros. En caso que querramos evitar ese comportamiento podemos nutrirnos de la siguiente funcion.
KEEPFILTERS: Changes the CALCULATE and CALCULATETABLE function filtering semantics.

```
Example:
CALCULATE ( 
	[Measure], 
	KEEPFILTERS ( Table[Color] = "Red" ) 
)
```

**Preguntar por valores en lista (IN en SQL)**

Existen dos funciones para preguntar por valores en una lista o tabla de una columna.
TREATAS: Treats the columns of the input table as columns from other tables.For each column, filters out any values that are not present in its respective output column.
```
Example:
CALCULATE(
	SUM(Table[Column1]),
	Table[Column2] IN {"value1", "value2", "value3"} 
)
CALCULATE(
	SUM(Table[Column1]),
	TREATAS ( 
		{"value1", "value2", "value3"} ,
		Table[Column2]
	)
)
CALCULATE(
	SUM(Table[Column1]),
	( Table[Column2], Table[Column3] ) IN { ( "value1", "value2" ), ( "value1.2", "value2.2" ) }
)
```

**Obtener total independiente del calculo de filas**

Vamos a aprovecharnos de la funcion HASONEVALUE para calcular un valor Total que es diferente al que se ejecuta en cada fila.
En este ejemplo la variable __rows retornara una suma y la variable __avg retornara el promedio de esa suma. 
La idea es que para cada fila se haga un calculo y que para el total de la fila o matriz se muestre un calcuo diferente al de las filas.
Esto se puede combinar con cualquier tipo de funcion y calculo, incluso con calculos que retornan tipos distintos de datos.
HASONEVALUE: Returns true when there’s only one value in the specified column.
```
Example:
VAR __rows = SUM(Table[Column1])
VAR __avg = DIVIDE(__rows, COUNTROWS(VALUES(Table[Column2])) )

RETURN 
IF( 
    HASONEVALUE(Table [Column2]) && 
    NOT(ISBLANK( __rows )), 
    __rows,
    __avg
) 
```

**Funcion FORMAT()**

Converts a value to text in the specified number format.
Returns a string containing Value formatted as defined by Format. The result is always a string, even when the value is blank.
Para muchos mas ejemplos visitar: https://dax.guide/format/
```
FORMAT( value, format, LocaleName(opcional) )

Example:
FORMAT( 0.42, "Percent" )                 -- Devuelve 42.00%
FORMAT( 0.42, "#,0.00%" )                 -- Devuelve 42.00%
FORMAT( DATE( 1997, 1, 9 ), "yyy-mm-dd" ) -- Devuelve 1997-1-9

```

**Top n % Share**

Vamos a ver como calcular el share de un Ranking, que significa esto? Calcular por ejemplo el % que representa la venta de los 10 productos mas vendidos, por ejemplo... Pueden ser 10, 5, 3, lo que quieran.
Lo primero que haremos sera crear una medida con el ranking, esa es sencilla. En el ejemplo lo haremos con los productos (Product Name) y sus ventas (Sales)

```
Rank Sales = 
RANKX(
    ALL( 'Product'[Product Name] ), 
    [Sales]
)

```
Luego crearemos una tabla "virtual" la cual contendra los nombres de productos, sus ventas y el ranking.
Esta tabla se la pasaremos como parametro a un CALCULATE() para que calcule las ventas de la misma. Es decir a sales, lo calcularemos dentro de un contexto de filtro dado por esta tabla, que tenga el ranking de productos <= al numero deseado.
El argumento que acepta CALCULATE es una table, o sea una lista de valores. La tabla provista como filtro define la lista de valores que van a ser visibles durante la evaluacion de la expresion. Es por esto que al pasarle como parametro la tabla "_ranktable", los unicos valores de Product Name visibles para la expresion, son aquellos que cumplen con la condicion Rank Sales <= n

```
VAR _ranktable =
FILTER(
    ADDCOLUMNS(
        ALL('Product'[Product Name]),      -- A la columna Product Name
        "@sales", [Sales],                 -- Le agregamos las ventas por producto 
        "@rank",  [Rank Sales]             -- Luego agregamos el Ranking
    ),
    [@rank] <= n                           -- Y filtramos la tabla que devuelve el ADDCOLUMNS para que contenga
)                                          -- solo el ranking <= al nro deseado

VAR _salestopn =
    CALCULATE(
        [Sales],
        _ranktable                         -- Le pasamos la tabla calculada al calculate para que calcule las ventas en ese 
    )                                      -- contexto de filtro donde [Product Name] IN { Productos pertenecientes a rank <= n }
	
VAR _totalsales =
    CALCULATE(                             -- Aca calculamos el valor total de las ventas para todos los productos
        [Sales],
        ALL('Product'[Product Name] )
    )
	
VAR _result =
    DIVIDE(                                -- Luego realizamos la division entre las ventas top n y las totales
        _salestopn,                        -- que nos dara como resultado lo que se llama Sales Top n Share
        _totalsales,
        0
    )
	
RETURN
    _result
	
```

---

### Clásicas de Fechas

*NOTA: Es necesario una tabla fecha para la resolución de la mayoría de las mencionadas.*

**Último Mes de fecha actual**

```
LastMonth =  Date(YEAR(EDATE(today(),-1)),MONTH(EDATE(today(),-2)),1)
```

**Valor agregado de una columna en su última fecha**
```
Valor de ultima fecha = 
CALCULATE (
    SUM ( 'Table'[Columna] ),
    LASTDATE ( 'TablaFecha'[Fecha] )
)
```


**Acumuladas to-date. Total or Date**

- TOTALYTD or DATESYTD: acumulado (cuatrimestre, mes, dia) anual de valores que se reinicia por año.
- TOTALQTD or DATESQTD: acumulado cuatrimestral (mes, dia) de valores que se reinicia por cuatrimestre.
- TOTALMTD or DATESMTD: acumulado mensual de valores por día que se reinicia por mes.

```
Examples:
CALCULATE(
	[Medida];
	DATESYTD ( DateTable[DateColumn] )
)

TOTALYTD (
	[Medida];
	[DateColumn];
	Opcional (Tabla Filtro);
	Opcional (Fecha Fin Año ejemplo: "6/30")
)
```

*NOTA: Solo TOTALYTD tiene un cuarto parametro para setear cual es la fecha considerada última del año para casos que el acumulado sea por año fiscal y no año calendario.*

**Running total**

En caso que querramos que nuestra medida acumule sobre todas las fechas sin importar un corte tenemos la siguiente función:

```
Example:
CALCULATE (
	[Medida];
	FILTER (
		ALLSELECTED( Table );
		Table[Date] <= MAX ( Table[Date] )
	)
)
```
	
**Otras Funciones útiles**

- PREVIOUSMONTH (Dates) devuelve los valores del mismo día mes anterior, si no hay filtro de día interpreta diferencias en 29 30 y 31 de cierre de mes.
- SAMEPERIODLASTYEAR (Dates) devuelve misma fecha del año anterior. Va fila por fila viendo la fecha, si es 20/4/2018 busca el valor para 20/4/2017.
- ENDOFMONTH (Date) devuelve una fecha con el último día del mes en el que estoy parado. Diferencia entre 29 30 y 31.
- EOMONTH (Dates, number) devuelve el último día del mes de la última fecha de las pasadas por parametros. El segundo parametro es para que devuelve N meses más o menos del último.
- DATESINPERIOD (dates,start_date,number_of_intervals,interval) Devuelve una tabla que contiene una columna de fechas que empieza por start_date y sigue hasta el valor de number_of_intervals especificado.
- ***Días habiles*** Proximamente... Es necesario contar con una tabla de domingos y feriados o de días hábiles para modelar esta solución.

---

### DAX Studio Basics

Es una herramienta que nos permite conectarnos a un modelo de datos tabular. El mismo tiene conexión a un Analysis services on premise, en azure o en power bi premium. La herramienta puede conectarse a un power bi desktop abierto al mismo tiempo en el sistema operativo.
Nos permite ejecutar consultas DAX sobre el modelo conectado o conocer detalles sobre el engine vertipaq y DMV.
Las ejecuciones devueltas por EVALUATE serán funciones de tablas.

```
Testear una medida en DAX Studio
EVALUATE
ROW( 
	"Medida", 
	[NewMedida]
)

Testear una columna en DAX Studio
EVALUATE
ADDCOLUMNS(
	Table,
	[NewColumn]
)
```

**Chequeo de duplicados**

Útil para revisar datos en DaxStudio dado que devolvería una tabla.

```
Example:
EVALUATE
FILTER (
    ADDCOLUMNS (
        SUMMARIZE ( Table, Table[Column] ),
        "@CantDuplicados", CALCULATE ( COUNTROWS ( Table ) )
    ),
    [@CantDuplicados] > 1
)
```

---

### Parches de modelado en medidas

**Usar una relación inhabilitada (linea de puntos)**

Si por un sencillo número hay que crear una relación que rompe el modelado, mejor no crearla y calcularlo con una medida.

```
Example: donde las columnas son las que hacen la relación
CALCULATE( 
	[Medida] , 
	USERRALATIONSHIP ( Table[ColumnaRelacionada], Table2[ColumnaRelacionada]) 
)
```

**Utilizar una navegabilidad crossfilter**

Si por alguna razón necesitamos que una relación propague sus filtros de IDA Y VUELTA, lo cual no se recomienda, mejor que se propague para una SOLA medida y no para todo el modelo.
```
Example:
CALCULATE( 
	SUM(Tabla1[Columna]),
	CROSSFILTER(Tabla1[ColumnaRelacionada], Tabla2[ColumnaRelacionada], BOTH)
)
```

**Filtrar una dimensión por un hecho**

Para llegar a una dimensión bajo un contexto de hecho podemos construir una DAX. La idea es si tenemos un hecho con dos dimensiones y necesitamos que uno filtre al otro bajo el hecho (como ver los productos vendidos de una tienda filtrada) podemos construir una DAX que nos indique si está pasando por el hecho. Si el resultado es 1 estaría pasando y podemos filtrar de dicha forma bajo nivel visual a nuestra visualización.

```
Example:
HasHecho? = INT ( NOT ISEMPTY ( TablaHecho ) )
```

---