# sierra-db-as-promised

Simplifies getting a pg-promise Database object set up for connecting to a
Sierra database, using environment variables for the connection settings.




----

## How to use

```
npm install 'SydneyUniLibrary/sierra-db-as-promised#v2.0.0'
```

Create a .env file in the root directory of your project, like the following.

```
SIERRA_DB_HOST=sierra.library.edu
SIERRA_DB_USER=me
SIERRA_DB_PASSWORD=secret
```

> **Never** commit this .env file into a source control repository.

In your code, require the library.

```javascript
const sierraDb = require('@sydneyunilibrary/sierra-db-as-promised')()
```

> Note the parentheses after `require(…)`. These are necessary.

`sierraDb` is a [pg-promise Database object](http://vitaly-t.github.io/pg-promise/Database.html).




----

## Examples

See also https://github.com/vitaly-t/pg-promise/wiki/Learn-by-Example.


### Using positional parameters

See also https://github.com/vitaly-t/pg-promise/wiki/Learn-by-Example#single-parameter.

```javascript
async function example() {
  const patronRecordNum = 1436323
  let patron =
    await sierraDb.one(
      `
         SELECT p.*
           FROM patron_record AS p
                JOIN record_metadata AS pmd ON ( pmd.id = p.record_id )
          WHERE pmd.record_num = $1
      `,
      patronRecordNum
    )
  console.log(patron)
}
```

> Note that this uses the `one(…)` function, so it will throw if no patron record is found.
> Use `oneOrNone(…)` if you would want `patron` to be `undefined` instead of having an Error thrown.


### Using named parameters

See also https://github.com/vitaly-t/pg-promise/wiki/Learn-by-Example#named-parameters.

```javascript
async function example() {
  let checkouts =
    await sierraDb.any(
      `
         SELECT *
           FROM checkout
          WHERE ptype = $<ptype>
                AND due_gmt = $<dueDate>
      `,
      {
        ptype: 1,
        dueDate: new Date(2018, 1, 6, 4, 0, 0),
      }
    )
  console.log(checkouts)
}
```

> `any(…)` is the same as `manyOrNone(…)`. `checkouts` would be an empty Array if no checkouts are found.
> Use `many(…)` if you want an Error thrown if no checkouts are found.

> When using template strings, you can't use `${…}` for named parameters because that already means something in template strings.


### Doing a series of SQL commands

It's important use [pg-promise Task](http://vitaly-t.github.io/pg-promise/Database.html#task) if you want
to do a series of SQL commands. Task will get a connection and reuse that connection for each command.
Without using task, you will get and release a connection for each command.

See also https://github.com/vitaly-t/pg-promise/wiki/Learn-by-Example#tasks.

```javascript
async function example() {
  const patronRecordNum = 1470581;
  let checkouts =
    await sierraDb.task(async t => {
      let patronRecordMetadata =
        await t.one(
          "SELECT * FROM record_metadata WHERE record_type_code = 'p' AND record_num = $1",
          patronRecordNum
        )
      return t.many(
        'SELECT * FROM checkout WHERE patron_record_id = $1',
        patronRecordMetadata.id
      )
    })
  console.log(checkouts)
}
```

> Note that the use of `t` instead of `sierraDb`.



----

## Configuration

sierra-db-as-promised initialises the pg-promise library and creates a Database object on your behalf.
While you can customise both how it initialises the pg-promise library and how it creates the Database object,
but default it should just do the right thing for connecting to a Sierra database.

The only variable you are required to define is `host`. You will typically do that by defining `SIERRA_DB_HOST` in the .env file.

You are likely to want to also define `user` and `password`. You wll typically do that by defining `SIERRA_DB_USER` and `SIERRA_DB_PASSWORD` in the .env file.


### Configuring the Sierra connection

The combination of sierra-db-as-promised, dotenv and pg-promise gives you a number of methods to set the Sierra connection configuration variables. None of these methods are mutually exclusive. You could set some variables using one method and set other variables using another method. However each methods take precendence over other methods, menaing any particular variable will only be set by one of the methods.

#### Configuration variables

Variable | connectionObject | Environment        | .env file          | Environment | Fallback default
---------|------------------|--------------------|--------------------|-------------|------------------
host     | host             | SIERRA_DB_HOST     | SIERRA_DB_HOST     | PGHOST      | localhost
port     | port             |                    |                    |             | 1032
database | database         |                    |                    |             | iii
user     | user             | SIERRA_DB_USER     | SIERRA_DB_USER     | USER        |
password | password         | SIERRA_DB_PASSWORD | SIERRA_DB_PASSWORD | PGPASSWORD  | (\*1)
ssl      | ssl              |                    |                    |             | true
max      | max              | SIERRA_DB_MAX      | SIERRA_DB_MAX      |             | 5 (\*2)
min      | min              | SIERRA_DB_MIN      | SIERRA_DB_MIN      |             | 0

See (pg-promise's connection syntax)[https://github.com/vitaly-t/pg-promise/wiki/Connection-Syntax] for the definition of these variables.

The columns in the table above represent the methods that can be used to set the Sierra connection configuration variables. The methods in the columns to the left take precendence over the methods in the columns to the right. Effectively, reading a row from left to right, the variable will get the value from the first method that has defines that variable.

The second environment column exists because pg-promise also looks for values amongst the environment variables, but looks for generic variables. The environemnt variables specific to sierra-db-as-promised take precendence over the generic environemnt variables.

(\*1) If neither `SIERRA_DB_PASSWORD` nor `PGPASSWORD` are set, pg-promise will look for the password in
[`~/.pgpass`](https://www.postgresql.org/docs/9.4/static/libpq-pgpass.html).

(\*2) The PostgreSQL users created by Sierra are limited to a maximum of 5 concurrent connections. Don't set `max` to be more than 5.


#### .env file

A .env file in your project's base directory, where `package.json` is, will be loaded by [dotenv](https://github.com/motdotla/dotenv)
as part of requiring sierra-db-as-promised. Anything in here, including other variables than those used by this library, will be added
to your envrionment and will be accessible via `process.env`.

When dotenv loads the .env file, it will not change any environment variables that are already defined. You can't use
to it redefine something like `PATH`. This also means that if you set `SIERRA_DB_HOST`, for example, in your environment
before running your code, the value you set will take precedence over the value in .env.


#### connectionObject

> Avoid using this method. Prefer to use a .env file.

You can create a
[pg-promise connection configuration object](https://github.com/vitaly-t/pg-promise/wiki/Connection-Syntax#configuration-object)
and pass it to sierra-db-as-promised.

```javascript
const connectionObject = { /* … */ }
const sierraDb = require('@sydneyunilibrary/sierra-db-as-promised')({ connectionObject })
```

Parameters set in `connectionObject` have the highest precendence, overriding any environment variables and the .env file.
They even override parameters hardcoded into sierra-db-as-promised such as `port` and `database`.


#### Example of using multiple methods

Assume that you have this .env file.

```
SIERRA_DB_HOST=sierra.library.edu
SIERRA_DB_USER=me
SIERRA_DB_PASSWORD=secret
```

And assume that you have this `test.js`.

```javascript
const sierraDb = require('@sydneyunilibrary/sierra-db-as-promised')()

console.log(
  'host=%s;user=%s;password=%s',
  sierraDb.$cn.host,
  sierraDb.$cn.user,
  sierraDb.$cn.password,
)
```

Running `node test.js` will output:

```
host=sierra.library.edu
user=me
password=secret
```

All 3 of these variables are defined in the .env file, so sierra-db-as-promises uses the values in that.

However running `SIERRA_DB_HOST=training-sierra.library.edu node test.js` will output:

```
host=sierra-sierra.library.edu
user=me
password=secret
```

While all 3 variables are still defined in the .env file, the `SIERRA_DB_HOST` environment variable set at the command line takes precendence and overrides the value in the .env file. Notice that the other 2 variables are still taken from the .env file.


### Configuring pg-promise

By default, sierra-db-as-promised initialises pg-promise with its default values. It effectively does a `require('pg-promise')()`.

You can pass a pg-promise `initOptions` object to sierra-db-as-promised if you want to set any of 
[pg-promise's library initialization options](http://vitaly-t.github.io/pg-promise/module-pg-promise.html).

```javascript
const initOptions = { /* … */ }
const sierraDb = require('@sydneyunilibrary/sierra-db-as-promised')({ initOptions })
```

You can pass both a `connectionObject` and an `initOptions` to sierra-db-as-promised.

```javascript
const initOptions = { /* … */ }
const connectionObject = { /* … */ }
const sierraDb = require('@sydneyunilibrary/sierra-db-as-promised')({ initOptions, connectionObject })
```




----

## License

Copyright (c) 2017  The University of Sydney Library

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
