# pyspark-regression
A tool for regression testing Spark Dataframes in Python.

## Installation
### Via pip
You can install via pip:
```bash
pip install pyspark-regression==1.0
```
**Note:** This requires a working intallation of Spark 3+ and `pyspark>=3`.

### Via Git
To install via git:
```bash
git clone https://github.com/forrest-bajbek/pyspark-regression.git
cd pyspark-regression
pip install .
```
**Note:** This requires a working intallation of Spark 3+ and `pyspark>=3`.

### Via Docker
To build and then test the Docker Image:
```bash
git clone https://github.com/forrest-bajbek/pyspark-regression.git
cd pyspark-regression
make test
```

Images in Docker Hub coming soon.

## What is a Regression Test?
A [Regression Test](https://en.wikipedia.org/wiki/Regression_testing) ensures that changes to code only produce expected outcomes, introducing no _new_ bugs. These tests are particularly challenging when working with database tables, as the result can be too large to visually inspect. When updating a SQL transformation, Data Engineers must ensure that no rows or columns were unintentionally altered, even if the table has 300 columns and 400 billion rows.

`pyspark-regression` reduces the complexity of Regression Testing for structured database tables. It standardizes the concepts and jargon associated with this topic, and implements a clean Python API for running regression tests against DataFrames in [Apache Spark](https://spark.apache.org/).

## Example
Consider the following table:
| id | name | price |
| - | - | - |
| 1 | Taco | 3.001 |
| 2 | Burrito | 6.50 |
| 3 | flauta | 7.50 |

Imagine you are a Data Engineer, and you want to change the underlying ETL so that:
1. The price for Tacos is rounded to 2 decimal places.
1. The name for Flautas is capitalized.

You make your changes, and the new table looks like this:
| id | name | price |
| - | - | - |
| 1 | Taco | **3.00** |
| 2 | Burrito | 6.50 |
| 3 | **Flauta** | 7.50 |

Running a regression test will help you confirm that the new ETL changed the data how you expected.

Let's create the old and new tables as dataframes so we can run a Regression Test:
```python
from pyspark.sql import SparkSession
from pyspark.sql.types import *
from pyspark_regression.regression import RegressionTest

spark = SparkSession.builder.getOrCreate()
spark.conf.set("spark.sql.shuffle.partitions", 1)

schema = StructType(
    [
        StructField("id", IntegerType()),
        StructField("name", StringType()),
        StructField("price", DoubleType()),
    ]
)

# The old data
df_old = spark.createDataFrame(
    [
        (1, 'Taco', 3.001),
        (2, 'Burrito', 6.50),
        (3, 'flauta', 7.50),
    ],
    schema=schema
)

# The new data
df_new = spark.createDataFrame(
    [
        (1, 'Taco', 3.00),  # Corrected price
        (2, 'Burrito', 6.50),
        (3, 'Flauta', 7.50),  # Corrected name
    ],
    schema=schema
)

regression_test = RegressionTest(
    df_old=df_old,
    df_new=df_new,
    pk='id',
)
```


`RegressionTest()` returns a Python dataclass with lots of methods that help you inspect how the two dataframes are different. Most notably, the `summary` method prints a comprehensive analysis in Markdown. Here's what happens when you run `print(regression_test.summary)`:
```
# Regression Test: df
- run_id: de9bd4eb-5313-4057-badc-7322ee23b83b
- run_time: 2022-05-25 08:53:50.581283

## Result: **FAILURE**.
Printing Regression Report...

### Table stats
- Count records in old df: 3
- Count records in new df: 3
- Count pks in old df: 3
- Count pks in new df: 3

### Diffs
- Columns with diffs: {'name', 'price'}
- Number of records with diffs: 2 (%oT: 66.7%)

 Diff Summary:
| column_name   | data_type   | diff_category        |   count_record | count_record_%oT   |
|:--------------|:------------|:---------------------|---------------:|:-------------------|
| name          | string      | capitalization added |              1 | 33.3%              |
| price         | double      | rounding             |              1 | 33.3%              |

 Diff Samples: (5 samples per column_name, per diff_category, per is_duplicate)
| column_name   | data_type   |   pk | old_value   | new_value   | diff_category        |
|:--------------|:------------|-----:|:------------|:------------|:---------------------|
| name          | string      |    3 | 'flauta'    | 'Flauta'    | capitalization added |
| price         | double      |    1 | 3.001       | 3.0         | rounding             |
```

The `RegressionTest` class provides low level access to all the methods used to build the summary:
```python
>>> print(regression_test.count_record_old) # count of records in df_old
3

>>> print(regression_test.count_record_new) # count of records in df_new
3

>>> print(regression_test.columns_diff) # Columns with diffs
{'name', 'price'}

>>> regression_test.df_diff.filter("column_name = 'price'").show() # Show all diffs for 'price' column
+-----------+---------+---+---------+---------+-------------+
|column_name|data_type| pk|old_value|new_value|diff_category|
+-----------+---------+---+---------+---------+-------------+
|      price|   double|  1|    3.001|      3.0|     rounding|
+-----------+---------+---+---------+---------+-------------+
```

This example is accessable from the module:
```python
from pyspark_regression.example import regression_test
print(regression_test.summary)
```

For more information on these methods, please see the docs (coming soon).