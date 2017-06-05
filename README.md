# Knex Workflow
A place for me to keep notes on how to set up knex in express server
References:
http://knexjs.org/#Migrations-CLI
https://github.com/hihi-2017/phase-2-boilerplate

## Install globally in your computer
`npm i -g knex`

## Install locally in your project
1. `npm i --save knex` 
2. in **package.json**, "scripts": {
    "knex": "knex"
  },
  You're saying that when I run `npm run knex` in terminal, I'm referring to the knex module.

## After installing the knex library, then install the appropriate database library
pg for PostgreSQL, mysql for MySQL or MariaDB, sqlite3 for SQLite3, or mssql for MSSQL.
`npm i --save-dev sqlite3` in our case

## Initialize the knex library
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

## Connect knex to server 
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

## Make table
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

## To see your database
1. In terminal
reference:https://sqlite.org/cli.html
`sqlite 3 dev.sqlite3`
Now you're in the program
`.schema`
TO see all your tables, these two are there by dafault
```
CREATE TABLE "knex_migrations" ("id" integer not null primary key autoincrement, "name" varchar(255), "batch" integer, "migration_time" datetime);
CREATE TABLE "knex_migrations_lock" ("is_locked" integer);
```
Then you will see other tables you created and migrated beneath these two.
```
CREATE TABLE "users" ("id" integer not null primary key autoincrement, "uname" varchar(255));
```
To see content of an individual table
`select * from users`
To exit from the program
`Control-D` or `.exit` or `.quit`

## Make seed
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


