# Snowflake Semi-Structred Data Functions
Implementation of `filter_array` and `transform_array` for Snowflake.

Operations are defined using [`expr_eval`](https://github.com/silentmatt/expr-eval/tree/master#expression-syntax) - see project for full list of operators and functions.


## Motivation

While we can filter and transform arrays using `FLATTEN` and `GROUP BY`, it is complicated, and hard to both write and understand.

For instance, let's assume `t1` is a table, and it has a column named `number_array`. If we'd like to filter it and keep only positive numbers, here is how it would look:

```SQL
select
    array_agg(value) as filtered
from t1, lateral flatten(input => t1.number_array)
where value > 0
GROUP BY seq
```

Not only is it hard to understand, it would also cause a bug where rows with no positive numbers would disappear (instead of returning an empty array).

Here is how the same thing can be achieved now:

```sql
select filter_array('x > 0', t1.number_array) AS filtered
from t1
```


## Usage
The files define two functions, with overloaded implementations:

* `filter_array(condition, array [, additional_context])`

* `transform_array(transformation, array [, additional_context])`

`condition` / `transformation` is a VARCHAR (string) defining the function to run over elements ("Lambda function"). It must be the same for all rows, as it is parsed only once. The array element is given as `x`, and additional variables can be defined using the optional parameter.

`array` is an `ARRAY`/`VARIANT` with the elements to be transformed/filtered.

`additional_context` (or `kwargs`) is an optional parameter. It is of type `OBJECT`/`VARIANT`, and defines additional variables to be used. Its usage is demonstrated in the example below. Unlike the transformation, which has to be constant for all rows, this can vary by row.



## Full Example
```sql
-- Table definition
WITH t1 AS (
    select PARSE_JSON(
    $$
    [
        {"int_value": 5, "should_keep": false},
        {"int_value": 6, "should_keep": true}
    ]
    $$
    ) AS MY_COL,
    10 AS VALUE_TO_ADD
)

-- Usage
select 
    MY_COL, 
    VALUE_TO_ADD,
    transform_array('x.int_value + y', MY_COL, {'y': VALUE_TO_ADD}) AS TRANSFORMED_COL,
    filter_array('x.should_keep', MY_COL) AS FILTERED_COL
from t1

```

|   MY_COL                                                                             |   VALUE_TO_ADD  |   TRANSFORMED_COL  |   FILTERED_COL                                |
|--------------------------------------------------------------------------------------|-----------------|--------------------|-----------------------------------------------|
|   `[ {"int_value": 5, "should_keep": false}, {"int_value": 6, "should_keep": true} ]`  |   `10`            |   `[ 15, 16 ]`       |   `[ {"int_value": 6, "should_keep": true } ]`  |


## Additional functions
In addition to the functions defined in `expr_eval`, these functions have been implemented:
* `object` - Creates an object, similar to `OBJECT_CONSTRUCT` in snowflake (e.g. `transform_array('object("name1", x.arg1, "name2", x.arg2)')`).
* `sorted` - Sorts an array.


## Implementation details
Other platforms, such as Spark and Presto/Trino, allow defining lambda functions for creating the transformations. In order to simulate this ability under Snowflake, we have turned to Javascript. However, Snowflake disallows running `eval` in UDFs ([and rightfully so!](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval#never_use_eval!)), which means we can't define lambda functions dynamically.

For this reason, we have turned to Matthew Crumley's library, [`expr_eval`](https://github.com/silentmatt/expr-eval/tree/master#expression-syntax). It allows us to parse a subset of Javascript operations, and execute them - without dynamically executing arbitrary code.

While currently only `filter_array` and `transform_array` are implemented, the underlying mechanism of parsing and executing operations in JS might prove useful in additional cases. It is easy to extract the function parsing the operation and reusing it in other UDFs.

## Security considerations

As stated, these functions avoid the use of `eval()`. While they indeed provide a safer alternative to it, they cannot guarentee the safety of running untrusted user input, and in particular untrusted function statements. Evaluation of arbitrary user expressions is not recommended.
