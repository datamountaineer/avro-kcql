[![Build Status](https://travis-ci.org/Landoop/avro-kcql.svg?branch=master)](https://travis-ci.org/Landoop/avro-kcql)

# avro-kcql
This is a small utility library allowing you to translate the shape of an Avro record.
Let's say we have the following Avro Schema:

```json
{
  "type": "record",
  "name": "Pizza",
  "namespace": "com.datamountaineer.kcql.avro",
  "fields": [
    {
      "name": "ingredients",
      "type": {
        "type": "array",
        "items": {
          "type": "record",
          "name": "Ingredient",
          "fields": [
            {
              "name": "name",
              "type": "string"
            },
            {
              "name": "sugar",
              "type": "double"
            },
            {
              "name": "fat",
              "type": "double"
            }
          ]
        }
      }
    },
    {
      "name": "vegetarian",
      "type": "boolean"
    },
    {
      "name": "vegan",
      "type": "boolean"
    },
    {
      "name": "calories",
      "type": "int"
    },
    {
      "name": "fieldName",
      "type": "string"
    }
  ]
}
```

using the library one can apply to types of queries:
* to flatten it
* to retain the structure while cherry-picking and/or rename fields
The difference between the two is marked by the **_withstructure_*** keyword.
If this is missing you will end up flattening the structure.
This library is dependant on **com.datamountaineer.kcql** hence you still have to provide a **'from topic'**

Let's take a look at the flatten first. There are cases when you are receiving a nested
avro structure and you want to flatten the structure while being able to cherry pick the fields and rename them.
Imagine we have the following Avro schema:
```
{
  "type": "record",
  "name": "Person",
  "namespace": "com.datamountaineer.kcql.avro",
  "fields": [
    {
      "name": "name",
      "type": "string"
    },
    {
      "name": "address",
      "type": {
        "type": "record",
        "name": "Address",
        "fields": [
          {
            "name": "street",
            "type": {
              "type": "record",
              "name": "Street",
              "fields": [
                {
                  "name": "name",
                  "type": "string"
                }
              ]
            }
          },
          {
            "name": "street2",
            "type": [
              "null",
              "Street"
            ]
          },
          {
            "name": "city",
            "type": "string"
          },
          {
            "name": "state",
            "type": "string"
          },
          {
            "name": "zip",
            "type": "string"
          },
          {
            "name": "country",
            "type": "string"
          }
        ]
      }
    }
  ]
}
```
Applying this SQL like syntax
```
SELECT 
    name, 
    address.street.*, 
    address.street2.name as streetName2 
FROM topic
```
the projected new schema is:
```
{
  "type": "record",
  "name": "Person",
  "namespace": "com.datamountaineer.kcql.avro",
  "fields": [
    {
      "name": "name",
      "type": "string"
    },
    {
      "name": "name_1",
      "type": "string"
    },
    {
      "name": "streetName2",
      "type": "string"
    }
  ]
}
```

There are scenarios where you might want to rename fields and maybe reorder them.
By applying this SQL like syntax on the Pizza schema

```
SELECT 
       name, 
       ingredients.name as fieldName, 
       ingredients.sugar as fieldSugar, 
       ingredients.*, 
       calories as cals 
FROM topic 
withstructure
```
we end up projecting the first structure into this one:

```json
{
  "type": "record",
  "name": "Pizza",
  "namespace": "com.datamountaineer.kcql.avro",
  "fields": [
    {
      "name": "name",
      "type": "string"
    },
    {
      "name": "ingredients",
      "type": {
        "type": "array",
        "items": {
          "type": "record",
          "name": "Ingredient",
          "fields": [
            {
              "name": "fieldName",
              "type": "string"
            },
            {
              "name": "fieldSugar",
              "type": "double"
            },
            {
              "name": "fat",
              "type": "double"
            }
          ]
        }
      }
    },
    {
      "name": "cals",
      "type": "int"
    }
  ]
}
```

## Flatten rules
* you can't flatten a schema containing array fields
* when flattening and the column name has already been used it will get a index appended. For example if field *name* appears twice and you don't specifically
rename the second instance (*name as renamedName*) the new schema will end up containing: *name* and *name_1*

## How to use it

```scala
import AvroKcql._
val record: GenericRecord = {...}
record.kcql("SELECT name, address.street.name as streetName FROM topic")
```
As simple as that!

## Query Examples
You can find more examples in the unit tests, however here are a few used:
* flattening
```
//rename and only pick fields on first level
SELECT calories as C ,vegan as V ,name as fieldName FROM topic

//Cherry pick fields on different levels in the structure
SELECT name, address.street.name as streetName FROM topic

//Select and rename fields on nested level
SELECT name, address.street.*, address.street2.name as streetName2 FROM topic
```
* retaining the structure
```
//you can select itself - obviousely no real gain on this
SELECT * FROM topic withstructure 

//rename a field 
SELECT *, name as fieldName FROM topic withstructure

//rename a complex field
SELECT *, ingredients as stuff FROM topic withstructure

//select a single field
SELECT vegan FROM topic withstructure

//rename and only select nested fields
SELECT ingredients.name as fieldName, ingredients.sugar as fieldSugar, ingredients.* FROM topic withstructure


```

## Release Notes


**0.1 (2017-05-03)**

* first release

### Building

***Requires gradle 3.4.1 to build.***

To build

```bash
gradle compile
```

To test

```bash
gradle test
```


You can also use the gradle wrapper

```
./gradlew build
```

To view dependency trees

```
gradle dependencies # 
```
