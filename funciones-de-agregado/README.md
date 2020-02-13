
- [Funciones de agregado](#funciones-de-agregado)
  - [Tipos](#tipos)
  - [¿De dónde salen las List(a)?](#de-dónde-salen-las-lista)
  - [¿Sobre qué se ejecutan?](#sobre-qué-se-ejecutan)
  - [¿Y WHERE qué?](#y-where-qué)
  - [Gotchas](#gotchas)
    - [Gotcha 1: NULL](#gotcha-1-null)
    - [Gotcha 2: List(a) is empty](#gotcha-2-lista-is-empty)
- [Agrupamientos de tuplas con GROUP BY](#agrupamientos-de-tuplas-con-group-by)
  - [Criterio de agrupamiento](#criterio-de-agrupamiento)
  - [¿Y el SELECT?](#y-el-select)
  - [Gotchas](#gotchas-1)
    - [Gotcha 1: NULL](#gotcha-1-null-1)
    - [Gotcha 2: Columnas, reductores y SELECT](#gotcha-2-columnas-reductores-y-select)
      - [Solución 1](#solución-1)
      - [Solución 2](#solución-2)
      - [Ambas soluciones](#ambas-soluciones)
      - [Corolario](#corolario)
- [HAVING](#having)
  - [WHERE, GROUP BY, HAVING, SELECT](#where-group-by-having-select)

# Funciones de agregado

Es como se suele llamar a las funciones que **reducen** una colección de valores a uno solo. En la literatura también aparecen como **funciones de columnas o colectivas**.

Tal y como vimos en clase, en realidad se trata de funciones de tipo `reduce` o `fold`, que de forma **muy** simplificada podemos asimilar a

```
reduce :: List(a) -> b
```

donde el tipo `a` puede o no ser igual al tipo `b`.

En ANSI/SQL podemos encontrarnos con:

- `AVG`: reduce a la media.
- `MAX`: reduce al máximo.
- `MIN`: reduce al mínimo.
- `SUM`: reduce a la suma.
- `COUNT`: reduce al número de elementos.

Podemos considerar `DISTINCT` como una función de agregado que reduce la duplicidad.

## Tipos

```
AVG :: List(Number) -> Number
MAX :: List(Number) -> Number
MIN :: List(Number) -> Number
SUM :: List(Number) -> Number
COUNT :: List(a)    -> Number
```

En `AVG`, `MAX`, `MIN` y `SUM` el tipo `a` es igual al tipo `b` y debe, además, ser númerico. En `COUNT`, sin embargo, `a` puede ser cualquier tipo válido en SQL, mientras que `b` debe ser numérico.

En el caso de `DISTINCT`, `a` y `b` pueden ser cualquier tipo válido en SQL pero tienen que ser el mismo (`a = b`).

```
DISTINCT :: List(a) -> a
```

## ¿De dónde salen las `List(a)`?

De los valores de las columnas de una tabla. Por ejemplo, supongamos que una columna contiene las `menciones` de un `hashtag`.

| id | hashtag          | menciones |
|----|------------------|-----------|
| 1  | `#AquiSufriendo` |  35       |
| 2  | `#LaCoruNeno`    |  140      |
| 3  | `#ImperativeSQL` |  3        |
| 4  | `#HaskellFTW`    |  1337     |

Si quisieramos saber qué `hashtag` tiene más menciones podríamos aplicar `MAX` a la columna `menciones`:

```sql
SELECT MAX(menciones) AS "menciones"
FROM HashtagTable;
```

## ¿Sobre qué se ejecutan?

Se ejecutan sobre columnas de **grupos de tuplas**. **Por defecto**, si no aparece ninguna cláusula `GROUP BY`, **toda la tabla** es un **único grupo de tuplas**. Sin embargo, si aparece un `GROUP BY`, se usará ese criterio para hacer diferentes agrupaciones de tuplas y posteriormente aplicar el _reductor_ a cada uno de los grupos.

Por lo tanto si sólo existe un único grupo, el resultado será una tabla con una sola tupla, mientras que si contamos con `n` grupos, obtendremos `n` tuplas.

## ¿Y `WHERE` qué?

`WHERE` se ejecuta **antes** que el _reductor_. Por tanto si _filtramos_ (recordad que `WHERE` es en realidad `filter`) acorde a un **predicado**:

1. Aplicamos el filtro (`WHERE`).
2. Sobre el resultado ya filtrado aplicamos el _reductor_.
3. El resultado será una tabla con una sola fila.

## Gotchas

### Gotcha 1: `NULL`

Los valores `NULL`. son eleiminado automáticamente antes de aplicar un _reductor_.

### Gotcha 2: `List(a)` is empty

Si la lista (`List(a)`) sobre la que queremos aplicar un _reductor_ está vacia:

- `COUNT` produce `0`.
- `AVG`, `MAX`, `MIN` y `SUM` producen `NULL`.
- **TODO**: ¿`DISTINCT` produce `NULL`?

# Agrupamientos de tuplas con `GROUP BY`

Tal y como comentábamos con anterioridad

> **Por defecto**, si no aparece ninguna cláusula `GROUP BY`, **toda la tabla** es un **único grupo de tuplas**.

Por lo tanto, `GROUP BY` permite agrupar tuplas acorde a un criterio. Estos grupos de tuplas pueden ser pensados como nuevas subtablas.

## Criterio de agrupamiento

Se usa una o varias columnas para agrupar, de forma que todas aquellas tuplas que tengan el **mismo valor para esa(s) columnas** formarán parte del mismo grupo/subtabla.

Pensemos en una tabla con todo el alumnado del Wirtz. Si agrupamos por la columna `curso`, todo el alumnado que pertenezca a un mismo curso (**mismo valor para la columna `curso`**) formarán parte del mismo grupo/subtabla.

## ¿Y el `SELECT`?

El `SELECT` se ejecuta sobre cada uno de los grupos/subtablas. Por tanto si tenemos 8 grupos como resultado de un `GROUP BY`, tendremos 8 tuplas en el resultado final tras la ejecución del `SELECT`.

## Gotchas

### Gotcha 1: `NULL`

Los `NULL` se consideran iguales entre sí y por lo tanto van todos a un mismo grupo/subtabla.

### Gotcha 2: Columnas, _reductores_ y `SELECT`

¿Qué pasa con este tipo de consultas?

```sql
SELECT
  name,
  classroom,
  MAX(grade) AS 'Nota máxima'
FROM Wirtz
GROUP BY classroom;
```

Suponiendo que hay:

- 900 alumnos/as en el Wirtz
- 35 clases en el Wirtz

analicemos el número de tuplas del resultado esperado por cada argumento del `SELECT`,

- `SELECT name` debería devolver tantas tuplas como alumnos hay en la tabla `Wirtz`, es decir 900 tuplas.
- `SELECT classroom ... GROUP BY classroom` debería devolver una tupla por cada clase, es decir 35 tuplas.
- `SELECT MAX(grade) AS 'Nota máxima' ... GROUP BY classroom` debería devolver una tupla por cada clase, es decir 35 tuplas.

¿Vemos el problema? ¿Cómo hacemos para que **SQL devuelva 900 tuplas y 35 tuplas a la vez**?

**NO PODEMOS**. De ahí el error.

#### Solución 1

Eliminamos `name` de la consulta

```sql
SELECT
  classroom,
  MAX(grade) AS 'Nota máxima'
FROM Wirtz
GROUP BY classroom;
```

#### Solución 2

Añadimos `name` a `GROUP BY`

```sql
SELECT
  name,
  classroom,
  MAX(grade) AS 'Nota máxima'
FROM Wirtz
GROUP BY classroom, name;
```

#### Ambas soluciones

En ambos casos devolvemos 35 tuplas (tantas como clases hay en el Wirtz).

#### Corolario

Si aparece una columna en un `SELECT ... GROUP BY` y no forma parte del `GROUP BY`, tenemos un problema. Pensemos en `name` en el ejemplo anterior.

La excepción a esta regla es que esa columna sea el argumento (`List(a)`) de un _reductor_ ya que entonces sí sería correcto. Pensemos en `MAX(grade) AS 'Nota máxima'` en ele ejemplo anterior, ya que `grade` aparece en el `SELECT` como `List(a)` de `MAX` y no aparece en `GROUP BY`.

# `HAVING`

Tras haber formado los **grupos/subtablas** con `GROUP BY`, descartar aquellos grupos que no cumplan el predicado del `HAVING`.

## `WHERE`, `GROUP BY`, `HAVING`, `SELECT`

**TODO**: Revisar esto. No está del todo claro.

Este es precísamente el orden de ejecución.

1. Ejecutamos el predicado del `WHERE` **en cada una de las tuplas**, filtrando aquellas que no lo cumplan.
2. Ejecutamos el `GROUP BY` **en cada una de las tuplas**, haciendo grupos/subtablas según el criterio especificado.
3. Ejecutamos el `HAVING` **una vez por cada grupo/subtabla**.
4. Finalmente ejecutamos el `SELECT` **una vez por cada grupo/subtabla**.
