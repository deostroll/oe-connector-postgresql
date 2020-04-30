## oe-connector-postgresql

This is modified version of loopback-connector-postgresql which is loopback compliant connector for PostgreSQL.
This version is named as oe-connector-postgresql and is maintained at [github location](https://github.com/EdgeVerve/oe-connector-postgresql)
Original loopback connector for PostgreSQL is maintained at [Github Location](http://docs.strongloop.com/display/LB/PostgreSQL+connector).


## Getting Started
In your application root directory, enter this command to install the connector:
```javascript
$ npm install oe-connector-postgresql --save
```
This will install the module from npm and add it as a dependency to the application’s package.json file.

## Design

Original Loopback connector for PostgreSQL has many limitations. Few of the limitations are
* It does not support Array types.
* It does not support JSON data type
*


Due to above limitations, oeCloud team has modified original loopback connector to support these features.

* Following Mapping between Loopback types and PostgreSQL types have been implemented
    * Array of numbers, string, date time, bool and object
    * Json data type (in postgresSQL it is called Jsonb)
    * Data type of id is chosen as string by default and if not supplied is auto-populated as uuid
* Support added to query on json data type and array and thus this way, user can query embedded models and also embedded object.
* Include clause, contains clause added to support array and json data types. So now we can query Customer model

```javascript
  address.city = ‘Bangalore’ where address is embedded model.
```

## Best Practices

* All models must have ‘strict’ to true. Meaning you can POST/GET only those properties you have defined in your model.
  Because PostgreSQL connector creates table when model is being created. After that if you add new fields, it will not know.
* Queries on object data type should be avoided. For example in one record, x.y is numeric 10 and in other record x.y = ‘abcd’
  and you query x.y = 10, PostgreSQL will through error as it cannot convert ‘abcd’ to numeric.
* You should always considered strongly typed properties of model.
* Model definition should be changed carefully. For example you have model X with 10 records in it now you want to
  add a field called 'name' as required field to it. For this you need to add a default value else model will not get
  updated.(for example please refer the **Limitations** section).
* In model definition, properties can include mapping to PostgreSQL column.
  The below example shows how a property "state" has its corresponding postgres column mapping configured inside it:

  ```javascript
  "properties" : {
      "state": {
          "type": "String",
          "required": false,
          "length": 20,
          "precision": null,
          "scale": null,
          "postgresql": {
              "columnName": "state",
              "dataType": "character varying",
              "dataLength": 20,
              "dataPrecision": null,
              "dataScale": null,
              "nullable": "YES"
          }
    }
  }
  ```
  Here the property "state" has a maximum length of 20 in general and the corresponding postgres mapping of the property also sets
  the maximum length to 20.But a mismatch in both the values will lead to ambiguity and user will get unexpected results.
  For example :

  ```javascript
  "properties" : {
      "state": {
          "type": "String",
          "required": false,
          "length": 20,
          "precision": null,
          "scale": null,
          "postgresql": {
              "columnName": "state",
              "dataType": "character varying",
              "dataLength": 10,
              "dataPrecision": null,
              "dataScale": null,
              "nullable": "YES"
          }
    }
  }
  ```
  Now the max length in general has value 20 whereas for postgres it is 10. A user will get error for data where length of "state"
  is greater than 10 for postgres database, but for any other database(e.g. mongo, etc.) there will be no error till the length is less
  than 20. This kind of configuration results in unexpected behaviour. So, one should carefully use this feature to avoid ambiguities.

## Limitations

* By default, **id** field of model is made of type **string**. It cannot be of type **number**. We will be supporting number data type also.

* **ANY** Data type will not be supported. This is true also for Array type. As a developer, all data types must be fixed.
If data type is of type **object**, queries may fail if data is not matching.
In example below, if you execute query where myobject.value : 1, it may fail because in first record, value is integer (1) while in other record value is string ("B").
As a developer, you should ensure that data is consistent.

```javascript
[
{
name : "a",
myobject : {
"key" : "a",
"value" : 1
},
id : 1
},
{
name : "b",
myobject : {
"key" : "b",
"value" : "B"
},
id  : 2
}
]
```
In same example above, if you execute query { "where" : {"and" : [{"id" : 1}, {"myobject.value": 1}]} }, it will work because record where id is 1 ensures that myobject.value is number field.

* When we query a table in EDB with a column name which is not present, it fetches all the records.
But unlike this mongo will fetch no records and give a blank array in response.

Example :
Let's have a model with the following schema

```
{
    "name" : "Customer",
    "base" : "BaseEntity",
    "plural" : "Customers",
    "description" : "EDB Updated Customer Model",
    "properties" : {
        "name" : {
	    "type" : "string"
	    },
        "age" : {
            "type" : "number"
        }
    }
}
```

So if we fire a query with a filter on column "names" which is not present in "Customer" table :
```
{"where": {"names": "George"}}
```
Mongo will fetch blank result : [] whereas,
EDB will fetch all records present in "Customer" table.


## Model Updation Limitations

Certain guidelines has to be followed while updating any model's schema/definition in EDB.

* **Column Addition** :
If any new column is added which is NOT NULL(i.e. required : true) its default value has to be provided.
Since data is already present for that model, addition of a new column means old data will also get updated
w.r.t the newly added column.So a default value is required.

For example :
Let's create a model with the following schema/definition
```
{
    "name" : "Customer",
    "base" : "BaseEntity",
    "plural" : "Customers",
	"description" : "EDB Customer Model",
    "properties" : {
        "name" : {
	        "type" : "string",
		    "required" : true
	    }
    }
}
```
Now update the "Customer" model, let's say the new schema/definition is
```
{
    "name" : "Customer",
    "base" : "BaseEntity",
    "plural" : "Customers",
	"description" : "EDB Updated Customer Model",
    "properties" : {
        "name" : {
	        "type" : "string",
		    "required" : true
	    },
		"age" : {
	        "type" : "number",
		    "required" : true,
			"default" : 10
	    }
    }
}
```
Here we need to add default value to newly added property/column "age" because all the previously stored
entries in "Customer" model/table will be implicitly updated w.r.t "age" column.


* **Primary Key Deletion/Rename** :

Primary key cannot be renamed once created.
Let's create a model with the following schema/definition
```
{
    "name" : "Customer",
    "base" : "BaseEntity",
    "plural" : "Customers",
	"description" : "EDB Updated Customer Model",
    "properties" : {
        "address" : {
	    "type" : "string"
	    },
		"ck" : {
	    "type" : "string",
		"id" : true
	    }
    }
}
```
Let's update the model by deleting the primary key "ck", now the schema is
```
{
    "name" : "Customer",
    "base" : "BaseEntity",
    "plural" : "Customers",
	"description" : "EDB Updated Customer Model",
    "properties" : {
        "address" : {
	    "type" : "string"
	    }
    }
}
```
This will throw an error : "column "id" does not exist", whenever there is a POST on this model.
So we have to provide the same primary key in this case "ck" every time we update the model.

**Note** : "id" is the default primary key.

Same problem will happen if we provide a new primary key, lets update the model/table with the following schema.
```
{
    "name" : "Customer",
    "base" : "BaseEntity",
    "plural" : "Customers",
	"description" : "EDB Updated Customer Model",
    "properties" : {
        "address" : {
	    "type" : "string"
	    },
        "newId" : {
            "type" : "string",
            "id" : true
        }
    }
}
```
This will throw an error : "column "newid" does not exist", whenever there is a POST on this model.

**Note** : If you create a model with default primary key "id", then also the above guidelines has to be followed.
The primary key cannot be renamed while model updation.While updating model which has the default primary key we
need not provide any primary key it will always be default.


## Automatic DB Creation

When the initialize connection of DB is triggered from the entry in datasources.json.
Properties 'database' need to set and 'enableDbCreation' need to be set to true.
Example settings:

```json
"exampleDS": {
    "name": "exampleDS",
    "connector": "postgresql",
    "host": "localhost",
    "port": 5432,
    "database": "pgDB",
    "username": "postgres",
    "password": "postgres",
    "enableDbCreation" : true
}
```

**NOTE**: This has only been done for testing and CI / CD purpose. In production, you should always have predefined database.
Database has large number of parameters to fine tune performance. In production, you should not depend on default parameters.
Also, application user will not have permissions to create database.
Settings with properties of host, port, username, password, database are given importance over the URL property. URL property parsing is not done.

## Multiple Versions of Apps running in parallel connected to same Postgres

### Dropping unsued columns

By default the postgres connector will allow to run multiple versions of same application connected to same Postgres.
With this, if there is any change in the Model properties difference from one version to another, the connector will
not delete difference columns of the particular version.

If there is a requirement to disable this functionality, disable it by using the property `'dropUnusedColumns'`
setting it to true.

## Sequences

This connector supports the creation of sequences in the database. Sequences in postgres can be implicitly defined using the SERIAL or BIGSERIAL postgres datatypes, or, they can be created independently whereby it can be consumed by various database objects. This connector supports both flavours.

This connector supports three different types of sequences.

1. Standard Sequence
2. Simple Sequence
3. Complex Sequence

#### General note

Only properties which are meant to be an instance's unique identifier can be defined as a sequence field. This is due to a limitation in the datasource juggler. Hence in the property definition they must be defined as `"id": true`. 

Refer example below.

### Standard sequence

This is the traditional flavor of SERIAL or BIGSERIAL. During automigrate the create scripts for a table are generated with this as the datatype against the corresponding column. Postgres automatically generates a sequence object and associates with the table column.

To specify sequence field:

```
"properties" : {
    "customerId": {
        "type": "number",
        "postgresql" : {
            "sequence" : {
                "type" : "standard",
                "bigSerial": false
            }
        },
        "id": true
    },
    "name" : "string"
}
```

> **Note**: This generates the following create script during automigrate.
> ```
> create table foo (
>     customerid SERIAL,
>     name VARCHAR
> )
> ```

### Simple Sequence

This generates a sequence object and associates with the corresponding table field in the database. The sequence object can later be associated with other tables in the database if required. Therefore most features of a typical postgres sequence is supported. For more information about postgres sequence objects and how they can be created please refer postgres documentation.

Below are properties meant to define a postgres simple sequence.

```JSON
{
  "name": "seq_name", // sequence name - required
  "incrementBy": 1,
  "minValue": false,  // number or boolean
  "maxValue": false,  // number or boolean
  "startFrom": 1,     // number
  "cache": 1,         // no. of sequences to cache, boolean (false) or a number. 1 is no cache. Min value - 1
  "cycle": false,     // restart once the seq reaches its upper bound.
}
```

E.g
```JSON
{
    "name" : "reservation",
    "properties" : {
        "reservationId": {
            "type": "number",
            "postgresql": {
                "sequence": {
                    "type": "simple",
                    "name": "reservation_sequence",
                    "incrementBy": 3,
                    "maxValue": 1999364,
                    "cycle" : true,
                    "startFrom": 10021
                }
            },
            "id": true
        },
        "firstName": "string",
        "lastName":"string"
        }
    }
}

```

> **Note**: Other than the required fields other properties will takeup default values as indicated above.

### Complex Sequence

In many systems, for e.g. e-commerce applications, the order numbers may have a prefix followed by a defined sequence (with left zero-padding usually). This is supported via means of a complex sequence. It is an extension of a simple sequence. Only additional required parameters are `length`, `prefix`. These both properties determine the padding length.

E.g.
```json
{
    "name": "reservations",
    "properties" : {
    "reservationId" : {
        "type": "string",
        "postgresql": {
          "sequence": {
            "type": "complex",
            "prefix": "LMB",
            "name": "reservation_sequence",
            "length": 10,
          }
        },
        "id": true
      },
      "firstName": "string",
      "lastName" : "string"
    }
}
```
> **Note**: Other than the required fields other properties will takeup default values as indicated in a simple sequence.
