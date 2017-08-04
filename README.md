# sierra-db-as-promised

Simplifies getting a pg-promise Database object set up for connecting to a
Sierra database, using environment variables for the connection settings.




## How to use

```
npm install 'SydneyUniLibrary/sierra-db-as-promised#v1'
```

Create a .env file in the root directory of your project, like the following.

```
SIERRA_DB_HOST=sierra.library.edu
SIERRA_DB_USER=me
SIERRA_DB_PASSWORD=password
```

In your code:

```javascript
const sierraDb = require('@sydneyunilibrary/sierra-db-as-promised')()
```

> Note the parentheses after `require(…)`. These are necessary.




## Examples

`sierraDb` is a [pg-promise Database object](http://vitaly-t.github.io/pg-promise/Database.html).


### Using positional parameters

```javascript
async function testPatronRecord() {
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

```javascript
async function testCheckout() {
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

```javascript
async function testGetCheckouts2() {
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




## Advanced configuration


### Configuring pg-promise

TODO


### Configuring the Sierra connection

TODO




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
