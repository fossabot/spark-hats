# spark-hats

Spark "**H**elpers for **A**rray **T**ransformation**s**"

This library extends Spark DataFrame API with helpers for transforming fields inside nested structures and arrays of
arbitrary levels of nesting.

## Usage

Reference the library

### Scala 2.11
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/za.co.absa/spark-hats_2.11/badge.svg)](https://maven-badges.herokuapp.com/maven-central/za.co.absa/spark-hats_2.11)

```
groupId: za.co.absa
artifactId: spark-hats_2.11
version: 0.2.0
```

### Scala 2.12
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/za.co.absa/spark-hats_2.12/badge.svg)](https://maven-badges.herokuapp.com/maven-central/za.co.absa/spark-hats_2.12)

```
groupId: za.co.absa
artifactId: spark-hats_2.12
version: 0.2.0
```

Please, use the table below to determine what version of spark-hats to use for Spark compatibility.

| spark-hats version | Scala version |  Spark version  |
|:------------------:|:-------------:|:---------------:|
|       0.x.x        |  2.11, 2.12   |     2.4.3+      |

Import the extensions into your scope.

```scala
import za.co.absa.spark.hats.Extensions._
```

## Methods

The usage of nested helper methods is shown on this simple example that contains an array of struct fields.

```scala
scala> df.printSchema()
root
 |-- id: long (nullable = true)
 |-- my_array: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- a: long (nullable = true)
 |    |    |-- b: string (nullable = true)
       
scala> df.show(false)
+---+------------------------------+
|id |my_array                      |
+---+------------------------------+
|1  |[[1, foo]]                    |
|2  |[[1, bar], [2, baz], [3, foz]]|
+---+------------------------------+
```

### Turning on the extensions

To use the extensions you need to add this import to your Spark application or shell:
```scala
import za.co.absa.spark.hats.Extensions._
```

### Add a column
The `nestedWithColumn` method allows adding new fields inside nested structures and arrays.

The addition of a column API is provided in two flavors: the basic and the extended API. The basic API is simpler to
use, but expressions it expects cannot reference columns on parent array levels. Here is an example of the basic add
column API:

```scala
scala> df.nestedWithColumn("my_array.c", lit("hello")).printSchema
root
 |-- id: long (nullable = true)
 |-- my_array: array (nullable = true)
 |    |-- element: struct (containsNull = false)
 |    |    |-- a: long (nullable = true)
 |    |    |-- b: string (nullable = true)
 |    |    |-- c: string (nullable = false)

scala> df.nestedWithColumn("my_array.c", lit("hello")).show(false)
+---+---------------------------------------------------+
|id |my_array                                           |
+---+---------------------------------------------------+
|1  |[[1, foo, hello]]                                  |
|2  |[[1, bar, hello], [2, baz, hello], [3, foz, hello]]|
+---+---------------------------------------------------+
```

### Add column (extended)
The extended API method `nestedWithColumnExtended` works similar to the basic one, but allows the caller to reference
other array elements, possibly on different levels of nesting. The way it allows this is a little tricky.
The second parameter is changed from being a column to a *function that returns a column*. Moreover, this function has
an argument which is a function itself, the `getField()` function. The `getField()` function can be used in the
transformation to reference other columns in the dataframe by their fully qualified name.

Int the following example, a transformation adds a new field `my_array.c` to the dataframe by concatenating a root
level column `id` with a nested field `my_array.b`:

```scala
scala> val dfOut = df.nestedWithColumnExtended(newColumnName = "my_array.c", expression = getField =>
         concat(getField("id").cast("string"), getField("my_array.b"))
       )

scala> dfOut.printSchema
root
 |-- id: long (nullable = true)
 |-- my_array: array (nullable = true)
 |    |-- element: struct (containsNull = false)
 |    |    |-- a: long (nullable = true)
 |    |    |-- b: string (nullable = true)
 |    |    |-- c: string (nullable = true)

scala> dfOut.show(false)
+---+------------------------------------------------+
|id |my_array                                        |
+---+------------------------------------------------+
|1  |[[1, foo, 1foo]]                                |
|2  |[[1, bar, 2bar], [2, baz, 2baz], [3, foz, 2foz]]|
+---+------------------------------------------------+
```

* **Note.** You can still use `col` to reference root level columns. But if a column is inside an array (like
`my_array.b`), invoking `col("my_array.b")` will reference the whole array, not an individual element. The `getField()`
function that is passed to the transformation solves this by adding a generic way of addressing array elements on arbitrary
levels of nesting.

* **Advanced Note.** If there are several arrays in the schema, `getField()` allows to reference elements of an array
if it is one of the parents of the output column.


### Drop a column
The `nestedDropColumn` method allows dropping fields inside nested structures and arrays.


```scala
scala> df.nestedDropColumn("my_array.b").printSchema
root
 |-- id: long (nullable = true)
 |-- my_array: array (nullable = true)
 |    |-- element: struct (containsNull = false)
 |    |    |-- a: long (nullable = true)

scala> df.nestedDropColumn("my_array.b").show(false)
+---+---------------+
|id |my_array       |
+---+---------------+
|1  |[[1]]          |
|2  |[[1], [2], [3]]|
+---+---------------+
```

### Map a column

The `nestedMapColumn` method applies a transformation on a nested field. If the input column is a primitive field the
method will add `outputColumnName` at the same level of nesting. If a struct column is expected you can use `.getField(...)` method to operate on its children.

The output column name can omit the full path as the field will be created at the same level of nesting as the input column.

```scala
scala> df.nestedMapColumn(inputColumnName = "my_array.a", outputColumnName = "c", expression = a => a + 1).printSchema
root
 |-- id: long (nullable = true)
 |-- my_array: array (nullable = true)
 |    |-- element: struct (containsNull = false)
 |    |    |-- a: long (nullable = true)
 |    |    |-- b: string (nullable = true)
 |    |    |-- c: long (nullable = true)

scala> df.nestedMapColumn(inputColumnName = "my_array.a", outputColumnName = "c", expression = a => a + 1).show(false)
+---+---------------------------------------+
|id |my_array                               |
+---+---------------------------------------+
|1  |[[1, foo, 2]]                          |
|2  |[[1, bar, 2], [2, baz, 3], [3, foz, 4]]|
+---+---------------------------------------+
```
