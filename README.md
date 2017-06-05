# Knex Workflow
A place for me to keep notes on how to set up knex in express server

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
> 
module.exports = {

    development: {
        client: 'sqlite3',
        connection: {
        filename: './dev.sqlite3'
        },
        useNullAsDefault: true
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
