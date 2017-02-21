---
layout: post
title: "Fencing Tokens in Node.js"
date: 2017-02-19T00:00:00+02:00
comments: true
tags: [node-js, distributed-systems]
---

In my experience, the average Node.js developer tends to be less concerned about thread safety issues and race conditions than the average developer who uses
more traditional languages such as Java, C# or C++. Perhaps the reason is that Node.js code is executed in a single-thread environment, so 
the risk of multiple threads mutating the same shared memory at once is avoided. However, this does not mean that we're entirely free from race 
conditions. For example, our database can be thought of as a shared state in which race condition may manifest.

<!-- more -->

![Header](/public/img/fencing-tokens/header.jpg)

A good start would be to recap the race condition definition. According to [wikipedia](https://en.wikipedia.org/wiki/Race_condition):

> A race condition [in a software system] is the behavior where the output of the code becomes dependent on the sequence or timing of other uncontrollable events

To make this more concrete, let's consider a simple example. Assume we write a service to manage an inventory of books. For simplicity, each
book only holds a title and an integer price. Our service should allow clients to update the price of a given book.

Even in this minimalist scenario, we may encounter a race condition if two requests to update the price of the same book arrive simultaneously. 
In this post we'd describe a method called *Fencing Token* that aims to prevent such race conditions. To set the expectations right, by
preventing the race condition I mean that it'll be possible to predict the final value of the book in face of concurrent updates in face of 
concurrent updates.

Our solution will use two common Node.js libraries:

1. [knex.js](https://knexjs.org) - a flexible SQL query builder
2. [bookshelf.js](https://bookshelfjs.org) - an ORM (object-relational mapper) built on knex.js. It provides goodies such as schema definition 
using migrations and lifetime callbacks

This post will be quite code heavy, and most of it is presented here. You can find the full version in [github](https://github.com/asafdav2/fencing-tokens).

Simultaneous updates
--------------------

We'd start by describing the main book model, using a knex.js migration file:

{% highlight JavaScript %}
exports.up = function(knex, Promise) {
  return knex.schema.createTable('books', table => {
    table.increments('id').primary();
    table.string('name');
    table.integer('price');
  });
};
{% endhighlight %}

And here's our minimal service that allows users to update a book's price.

{% highlight JavaScript %}
class BookService {
  updatePrice(bookId, price) {
    return Book.forge({id: bookId}).save({price: price});
  }
}
{% endhighlight %}

The following code illustrates two simultaneous updates to the same book. In this case, it's not deterministic what would be the final price; 
It could either be 20.0 or 30.0, depends on various timings that are out of our control.

{% highlight JavaScript %}
Promise.all([
  bookService.updatePrice(1, 20.0),
  bookService.updatePrice(1, 30.0),
])
{% endhighlight %}

Our aim therefore is to be able to determine the final book's price in face of simultaneous updates. 

Fencing Tokens
--------------

A *fencing token* is simply a positive number that increments its value. We maintain a specific token per each model. Before attempting to update a model instance, we *generate* a 
new token (read: increment the token), and include this token in the update attributes. Before applying an update request, we ensure that the provided token is not lower
than the current latest token. Else, the update request is discarded.

We start by creating a new table to hold the fencing tokens. We'll use one row for each model. Here's the migration file:

{% highlight JavaScript %}
exports.up = function (knex, Promise) {
  return knex.schema.createTable('fencing_tokens', table => {
    table.increments('id').primary();
    table.string('name');
    table.unique('name');
    table.integer('token').default(1);
  });
};
{% endhighlight %}

Next, the `FencingToken` model:

{% highlight JavaScript %}
const FencingToken = bookshelf.Model.extend({
  tableName: 'fencing_tokens'
});

FencingToken.generate = function (name) {
  return knex('fencing_tokens')
    .where('name', name)
    .increment('token')
    .returning('token')
    .then(result => result[0]);
};

function initialize(table) {
  return knex.raw(`
    INSERT INTO fencing_tokens (name, token) 
    VALUES ('${table}', 1) 
    ON CONFLICT DO NOTHING
  `);
}

FencingToken.initializer = function () {
  const tables = ['books'];

  return Promise.all(tables.map(initialize));
};
{% endhighlight %}

Two points are worth mentioning in the above code:

- We used knex.js [`increment`](http://knexjs.org/#Builder-increment) command to produce a query that increments the token value atomically 
(in oppose first query for the current value and then update the value in separate query. This code is not atomic and should not be used). 
The generated query looks like this:
{% highlight SQL %}
UPDATE fencing_tokens SET token = token + 1 WHERE name = '...'
{% endhighlight %}
- The `initialize` method is used to insert the first token for a model unless it already exists. To do it, we used PostgreSQL relatively-new 
`ON CONFLICT` clause (also known as `UPSERT`)

Now, let's look at the `Book` model:

{% highlight JavaScript %}
const Book = bookshelf.Model.extend({
  tableName: 'books',

  initialize: function () {
    this.on('updating', this.verifyFencingToken, this);
  },

  verifyFencingToken: function (model, attrs) {
    return knex('fencing_tokens')
      .where('name', 'books')
      .select('token')
      .then(rows => {
        const token = rows[0].token;
        if (token > attrs.token) {
            throw new Error('stale token provided');
        }
        delete this.attributes.token;
      });
  }
});
{% endhighlight %}

Notice the `verifyFencingToken` method that runs before updating a book. It checks the provided fencing token value against the latest fencing token value in the Database: if the provided value
is lower, we discard the update by throwing an exception. `delete this.attributes.token` is used so we don't actually attempt to write the token value into the books table.

Lastly, our modified `BookService` which handles updating a book price:

{% highlight JavaScript %}
class BookService {
  updatePrice(bookId, price) {
    return Promise.all([
      FencingToken.generate('books'),
      Book.forge({id: bookId}).fetch()
    ]).then(results => {
      const token = results[0];
      const book = results[1];
      return book
        .save({ price, token })
        .then(() => { price, token });
    });
  }
}
{% endhighlight %}

Notice how we first generate a new token before including it in the `save` arguments. 

These are all the building blocks we need. We are now ready to write a small stress test:

{% highlight JavaScript %}
 it ('determines the final price', function() {
   this.timeout(10000);

   const client = function(id) {
     return new Promise(resolve => {
       setTimeout(
         () => bookService
                .updatePrice(book.id, id)
                .then(resolve)
                .catch(resolve), 
         id
       );
     });
   };

   const clients = _.range(1000).map(id => client(id));
   return Promise.all(clients).then(results => {
     const expectedPrice = _.maxBy(results, 'token').price;

     return book.refresh().then((book) => {
       expect(book.get('price')).to.eql(expectedPrice);
     });
   });
 });
 {% endhighlight %}

In this test we create one book and 1,000 clients. Each client is delayed for some time and then tries to update the book's price. 
What this test proves is that we can anticipate the final price of the book by identifying the query with the maximum token value.

