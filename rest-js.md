---
layout: post
title: 'Creating a REST API in node.js'
category: computers
tags: [express, knex, nodejs, javascript]
date: 2017/4/19
---

Getting started with making web APIs can be confusing, even overwhelming at first. I'd like to share my process for creating APIs in Node.js.

# The server

First let's create a `package.json` and add a dependency:

```bash
cd rest_api
npm init -y
npm i express
```

Then let's create a super minimal 'hello world' with express and save it as `index.js`:

```javascript file=index.js
const express = require('express')

const app = express()

app.get('/', (req, res) => {
  res.send('Hello world!')
})

const port = process.env.PORT || 3000
app.listen(port, () => {
  console.log(`Express started on http://localhost:${port}\npress Ctrl+C to terminate.`)
})
```

- <small>I'm using ES2015 syntax here and throughout this guide, so if your node version doesn't support something, consider using [nvm][nvm] to get a more recent version.</small>

[nvm]: https://github.com/creationix/nvm

Express is imported, then an instance is created and saved as `app`.

`app.get()` tells the Express instance to listen for `GET` requests to the specified route; here it's just `/`. When this route is requested, the given callback is fired, in this case it sends the string "Hello world!" as a response.

Finally, `port` is declared as a variable which can be set from the environment, or otherwise defaults to `3000` if none is specified. Express then listens on that port, and after the server has finally started it uses `console.log` to print a message in the terminal.

Assuming everything went okay, you should then be able to run `node index.js` and point your browser at [http://localhost:3000/](http://localhost:3000/) and see the message "Hello world!"

---

Another way to test this instead of opening your browser is with either [cURL][curl] or [postman][postman]. cURL is great if you're the command-line junkie type (like me), postman is really pretty and slick but a bit too much hassle for me.

[curl]: https://curl.haxx.se
[postman]: https://getpostman.com

Once you have curl installed it should be as simple to get your Hello World in the shell with:

```bash
curl localhost:3000
```

## Request parameters

A server that only responds with the same thing every time isn't very fun though. Let's make a new route that allows us to say hello when passed a name:

```javascript file=index.js
app.get('/:name', (req, res) => {
  res.send(`Hello ${req.params.name}!`)
})
```

You'll want to add this before the `app.listen()` command so that it gets registered properly.

Then, with curl:

```bash
curl localhost:3000/foo
```

You should see "Hello foo!"

## Automatic restart

To get the above change working, you first have to stop the server and then restart it again, which can get fairly annoying after a few changes. [nodemon][nodemon] can be installed to watch for changes and restart automatically.

[nodemon]: https://nodemon.io/

```bash
npm i nodemon
```

Then alter your package.json so that it looks like this:

```diff
   "scripts": {
+    "start": "nodemon -w '*' -x 'node index.js'",
     "test": "echo \"Error: no test specified\" && exit 1"
   },
```

Now you can run `npm start` and it will use nodemon to watch for changes and restart the server.

# The database

Before making a database, you should first identify your data and what you'll be doing with it. I spent about 30 seconds debating what kind of data to use in this tutorial, and decided to do "something" with movies.

Among other things, a movie would have a title and a date it was released, and that seems like enough to work with for now.

---

I don't want to go into setting up and configuring a big fancy database, so we will [SQLite][sqlite], a [RDBMS][rdbms] that uses files to represent databases, and doesn't require a background processes to store or retrieve data. If you prefer PostgreSQL or MariaDB feel free to use those, you should only have to adjust a couple of things.

[sqlite]: https://sqlite.org/
[rdbms]: https://en.wikipedia.org/wiki/Relational_database_management_system

```bash
npm i sqlite3 knex
```

Rather than interacting with the database directly, we're going to use a query builder called [knex][knex]. knex allows you to write your queries in plain JavaScript, which provides an abstraction layer over your database driver. This makes it so your queries aren't necessarily tied to a specific database engine, and if you want to change to a different database later it's a much less difficult migration, sometimes only just a few lines.

[knex]: http://knexjs.org

`knex` has a really great cli. We can use it to create a default config:

```bash
npx knex init
```

Have a look at the file it creates at `knexfile.js`

In a new file, perhaps called `db.js`, save this:

```javascript file=db.js
const knex = require('knex')
const config = require('./knexfile.js')
const env = process.env.NODE_ENV || 'development'

module.exports = knex(config[env])
```

This creates and exports a single knex instance that we can re-use in other modules.

Now let's define a new table using the migrations cli:

```bash
npx knex migrate:make create_movies
```

This will create `migrations/<timestamp>_create_movies.js` with a few empty functions, change it so that it looks like this:

```javascript
exports.up = async knex => {
  return knex.schema.createTable('movies', t => {
    t.increments('id').primary()
    t.string('title')
    t.integer('released')
  })
}

exports.down = async knex => {
  return knex.schema.dropTableIfExists('movies')
}
```

Now, we can use the cli again to create our table:

```bash
npx knex migrate:latest
```

## Data methods

Now that we have a database and a table, we need some methods to populate the table with data.

```javascript file=movies.js
const db = require('./db')

async function create({ title, released }) {
  return db('movies').insert({ title, released })
}
```

This creates a function called `create` that accepts two parameters, `title` and `released`, which are purposely the same name as the field names. We then initialize a promise with knex to insert these, using [object shorthand notation][osn], which lets you use variable names for object keys.

[osn]: https://ariya.io/2013/02/es6-and-object-literal-property-value-shorthand

Now that we can create movies, we need a way to list them:

```javascript file=movies.js
async function list() {
  return db('movies').select('*')
}
```

To finish up, since this file won't directly consume these functions, we need to export them so they can be used as a module:

```javascript
module.exports = {
  create,
  list,
}
```

# Wiring it up

Now let's wire up our database to our server!

Going back to our `index.js`, let's import our new movies file and make a route to use the `list` method so that it looks like this:

```javascript file=index.js
const express = require('express')
const movies = require('./movies')

const app = express()

app.get('/api/v1/movies', async (req, res) => {
  res.json(await movies.list())
})

const port = process.env.PORT || 3000
app.listen(port, () => {
  console.log(`Express started on http://localhost:${port}\npress Ctrl+C to terminate.`)
})
```

Requesting the data is the easy part, with curl we can simply:

```bash
curl localhost:3000/api/v1/movies
```

which right now will return an empty JSON array.

Accepting `POST` requests is where things get a bit tricky...

To be able to properly create movies, we need to introduce a piece of express middleware:

- Express middleware are little modules that are run 'in the middle' of the request. They usually alter the request or response objects in some way. There are tons of middleware modules for express, and explaining them in-depth is a bit out of scope for this post, but if you'd like to read more, check out the [offical docs][middleware].

[middleware]: http://expressjs.com/en/guide/using-middleware.html

```bash
npm install --save body-parser
```

Then add it to the imports at the top:

```javascript file=index.js
const bodyParser = require('body-parser')
```

Then tell express to apply the middleware:

```javascript file=index.js
app.use(bodyParser.urlencoded({ extended: false }))
```

Finally we can add the route, and our `POST` body data will be available as `req.body`:

```javascript file=index.js
app.post('/api/v1/movies', (req, res) => {
  const { title, released } = req.body
  res.json(await movies.create({ title, released }))
})
```

The file should eventually look like:

```javascript file=index.js
const express = require('express')
const bodyParser = require('body-parser')
const movies = require('./movies')

const app = express()

app.use(bodyParser.urlencoded({ extended: false }))

app.get('/api/v1/movies', async (req, res) => {
  res.json(await movies.list())
})

app.post('/api/v1/movies', async (req, res) => {
  const { title, released } = req.body
  res.json(await movies.create({ title, released }))
})

const port = process.env.PORT || 3000
app.listen(port, () => {
  console.log(`Express started on http://localhost:${port}\npress Ctrl+C to terminate.`)
})
```

Using curl we can now insert data with:

```bash
curl -X POST -d 'released=2017' -d 'title=nothing good' localhost:3000/api/v1/movies
```

Which will return the new number of rows.

## Full CRUD support

Any good API endpoint usually provides [four methods][crud]:

- Create
- Read
- Update
- Delete

[crud]: https://en.wikipedia.org/wiki/Create,_read,_update_and_delete

We only have the first two, so let's finish it up and add the others.

Back in our `movies.js` let's add two more functions:

```javascript file=movies.js
async function update({ id, title, released }) => {
  return db('movies').update({ title, released }).where({ id })
}

async function del(id) {
  return db('movies').where({ id }).del()
}
```

Then we need to update our exports:

```diff
 module.exports = {
   create,
   list,
+  update,
+  del,
 }
```

Back in the `index.js` file we need to make two more routes:

```javascript file=index.js
app.put('/api/v1/movies', async (req, res) => {
  await movies.update(req.body)
  res.json(await movies.list())
})

app.delete('/api/v1/movies', async (req, res) => {
  await movies.del(req.body.id)
  res.json(await movies.list())
})
```

Now we can edit movies by passing field names as data pieces in curl:

```bash
curl -X PUT -d 'id=1' -d 'title=new title' localhost:3000/api/v1/movies
```

And delete them by passing the id:

```bash
curl -X DELETE -d 'id=1' localhost:3000/api/v1/movies
```

# Conclusion

We learned how to make a single endpoint respond to different actions and query a database with the appropriate methods.
