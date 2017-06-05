# Knex Workflow
A place for me to keep notes on how to set up knex in express server
References:
http://knexjs.org/#Migrations-CLI
https://github.com/hihi-2017/phase-2-boilerplate

### Install globally in your computer
`npm i -g knex`

### Install locally in your project
1. `npm i --save knex` 
2. in **package.json**, "scripts": {
    "knex": "knex"
  },
  You're saying that when I run `npm run knex` in terminal, I'm referring to the knex module.

### After installing the knex library, then install the appropriate database library
pg for PostgreSQL, mysql for MySQL or MariaDB, sqlite3 for SQLite3, or mssql for MSSQL.
`npm i --save-dev sqlite3` in our case

### Initialize the knex library
Migrations use a knexfile, which specify various configuration settings for the module. To create a new knexfile, run the following:
`knex init`
That command Created ./knexfile.js
```
  module.exports = {
    
    development: {
        client: 'sqlite3',
        connection: {
        filename: './dev.sqlite3'
        },
        useNullAsDefault: true (comment: this line is to address the warning in terminal "Knex:warning - sqlite does not support inserting default values. Set the `useNullAsDefault` flag to hide this warning. (see docs http://knexjs.org/#Builder-insert)."
        },
    production: {
        client: 'postgresql',
        connection: process.env.DATABASE_URL,
        pool: {
        min: 2,
        max: 10
        },
    migrations: {
    tableName: 'knex_migrations'
    }
   }
  };

```

### Connect knex to server 
In server.js, add 
```
var environment = process.env.NODE_ENV || 'development'
var config = require('../knexfile')[environment]
var knex = require('knex')(config)

```
Here you set up environment for knex, and require knex library in server.js
Then after you set up server with `var server = express()`
You give the knex database a name.
`server.set('whateveryouwannacallit', knex)`

### Make table
Once you have a knexfile.js, you can use the migration tool to create migration files to the specified directory (default migrations). Creating new migration files can be achieved by running:
`knex migrate:make migration_tablename`
This will create a migrations folder and a file. Open the newly created file and write out the migration
```

exports.up = function(knex, Promise) {
  return knex.schema.createTableIfNotExists('users', (table) => {
    table.increments('id').primary()
    table.string('name')
  })
};

exports.down = function(knex, Promise) {
  return knex.schema.dropTableIfExists('users')
};

```
Once you've written the migration, you can update the database matching your NODE_ENV (Not sure what this means) by running:
`knex migrate:latest`
Now the new table is in your database.
If you wanna take this table out of your database, rollback the last batch of migrations:
`knex migrate:rollback`

### To see your database
reference:
https://sqlite.org/cli.html
1. In terminal
Type
`sqlite 3 dev.sqlite3`
Now you're in the program. You can actually create table and insert seeds from here, but we are only using it to see the tables and their contents.
To see content of an individual table
`select * from users;`
**must inlcude the ; at the end, otherwise you will not see shit from your table.**
`.schema` this will show tables and their columns
these two are in schema by dafault
```
CREATE TABLE "knex_migrations" ("id" integer not null primary key autoincrement, "name" varchar(255), "batch" integer, "migration_time" datetime);
CREATE TABLE "knex_migrations_lock" ("is_locked" integer);
```
Then you will see other tables you created and migrated beneath these two.
```
CREATE TABLE "users" ("id" integer not null primary key autoincrement, "uname" varchar(255));
```
`.tables` this will show only names of the tables

To exit from the program
`Control-D` or `.exit` or `.quit`

2. In browser
There is a plug-in to see the tables in firefox/chrome browser I can't remember. Maybe I will add this later.

### Make seed
To create a seed file, run:
`knex seed:make populate_users`
Seed files are created in the directory specified in your knexfile.js for the current environment. A sample seed configuration looks like:
```
development: {
  client: ...,
  connection: { ... },
  seeds: {
      directory: './seeds/dev'
  }
}
```
In our case usually no seeds.directory is defined, by default files are created in ./seeds. Note that the seed directory needs to be a relative path. Absolute paths are not supported (nor is it good practice).
So now we have a file titled populate_users.js in seeds folder in root. We can edit it to fill it with our data.

```
exports.seed = function(knex, Promise) {
  // Deletes ALL existing entries
  return knex('users').del()
    .then(function () {
      // Inserts seed entries
      return knex('users').insert([
        {id: 1, name: 'Paul'},
        {id: 2, name: 'John'},
        {id: 3, name: 'Ringo'}
      ]);
    });
};
```
Now we can populate the table we specified in the seed file with seeds we wrote:
`knex seed:run`

## Now we have set up knex by itself, we need to link it to our website via routes 

### Write functions to get data from our knex database
1. Create ./server/db/db_functions.js (you can name is whatever you want, ususally it is named db.js, but I find it more clear if I call it db_functions.js)

Pause here. To be continued after we create the test and have the test fail properly.

### Set up tests to test the functions.
reference:https://github.com/hihi-2017/boilerplate-knex/tree/master/tests
1. Set up test environment
in ./knexfile.js add
```
 test: {
    client: 'sqlite3',
    connection: {
      filename: ':memory:'
    },
    useNullAsDefault: true
  }
```
2. Create test database
create ./tests/helpers/database-config.js
```
var knex = require('knex')
var config = require('../../knexfile').test

module.exports = (test, createServer) => {
  // Create a separate in-memory database before each test.
  // In our tests, we can get at the database as `t.context.db`.
  test.beforeEach(function (t) {
    t.context.connection = knex(config)
    if (createServer) t.context.app = createServer(t.context.connection)
    return t.context.connection.migrate.latest()
      .then(function () {
        return t.context.connection.seed.run()
      })
  })

  // Destroy the database connection after each test.
  test.afterEach(function (t) {
    t.context.connection.destroy()
  })
}
```
This will create a temporary database that has same tables and seeds as your knex database. You will use this test database to test your functions without making modifications to your real database.

3. Create test for database functions
create ./tests/db.test.js
```
// Note: we use AVA here because it makes setting up the
// conditions for each test relatively simple. The same
// can be done with Tape using a bit more code.

var test = require('ava')

var configureDatabase = require('./helpers/database-config')
configureDatabase(test)

var db = require('../server/db/db_functions') //whereever your functions are

test('getUsers gets all users', function (t) {
  var expected = 3
  return db.getUsers(t.context.connection)
    .then(function (result) {
      var actual = result.length
      t.is(expected, actual)
    })
})

test('getUsers gets a single user', function (t) {
  var expected = 'Paul'
  return db.getUser(1, t.context.connection)
    .then(function (result) {
      var actual = result[0].name
      t.is(expected, actual)
    })
})
```
4. Add test in scripts
in ./package.json, add 
```
 "scripts": {
    "test": "ava -v tests/**/*.test.js"
  },
```
This will run test for all files ending in .test.js in the tests folder. You might want to modify it during development to 
``` 
"scripts": {
    `"test-db":"ava -v tests/db.test.js" 
    }
```
So that you can run just the db tests without being distracted by other failing tests in the tests folder.
Now if we run the test, it should throw error saying deb.getUser is not a function because we have not written any function in the file.

### Write functions to get data from our knex database
Previously we did:
1. Create ./server/db/db_functions.js 
Now we have tests, we write the functions to make them pass the tests.
2. Write functions
```
    module.exports = {
      getUser: getUser,
      getUsers: getUsers
    }

    function getUsers (connection) {
      return connection('users').select()
    }

    function getUser (id, connection) {
      return connection('users').where('id', id)
    }
```
Now if we run the test again, we should have passing tests.

### Build an api on server to give knex data to client
1. Create ./server/routes/api.js (you can call it whatever)
```
const express = require('express')

const router = express.Router()
const dbFunctions = require('../db/db_functions')

// '/' == '/api', we set it in server later
// When we hit the '/api/users' route, we want info of all users to be sent back as a json file

router.get('/', (req, res) => {
  let db = req.app.get('knex-database')
  dbFunctions.gerUsers(db)
    .then(users => {
      res.json(users)
    })
})


router.get('/:id', (req, res) => {
  let connectAppToKnex = req.app.get('knex-database') //we set this in server.js with app.set('knex-database', knex)
  dbFunctions.getUser(req.params.id, connectAppToKnex) //we use getUser function from dbFunctions to get user info from knex database, then it is returned as result, which we named as userData
    .then(userData => {
      res.json(userData)
    })
})

module.exports = router
```
2. Set routes prefix in server.js
```
var api = require('./routes/api')
server.use('/api', api)
```
### Set up tests for routes
create ./tests/api/users.test.js
```
// Note: we use AVA here because it makes setting up the
// conditions for each test relatively simple. The same
// can be done with Tape using a bit more code.

// We use supertest here because we are testing api.

var test = require('ava')
var request = require('supertest')

var app = require('../../server')
var configureDatabase = require('./helpers/database-config')

//Here we are using the temporary database for testing again, so that actual database is not modified
configureDatabase(test, function (db) {
  app.set('knex-batabase', db)
})
```
Here we are setting the testing database to be used by the server, so 'knex-database' has to be same name as what we set knex to be in server.js. 
```
app.set('knex-database', knex)
```
So that when our test hits the server api routes, 
```
let db = req.app.get('knex-database')
```
Our app will get the test database as the database.

Ok now continue on the actual test code:
```
test.cb('GET /api/users gets all users', function (t) {
  var expected = 4
  request(app)
    .get('/api/users')
    .expect('Content-Type', /json/)
    .expect(200)
    .end(function (err, res) {
      if (err) throw err
      t.is(res.body.users.length, expected)
      t.end()
    })
})

//This is just an example
test.cb('POST api/users saves a user', (t) => {
  request(app)
    .post('/users')
    .send({name: 'Yoko'})
    .expect('Content-Type', /json/)
    .expect(201)
    .end((err, res) => {
      if (err) throw err
      return t.context.db('users')
        .select()
        .then((result) => {
          t.is(result.length, 5)
          t.is(result[26].name, 'Yoko')
          t.end()
        })
    })
})
```
