[![Build Status](https://api.travis-ci.org/tomwalder/php-gds.svg?branch=2.0)](https://travis-ci.org/tomwalder/php-gds)
[![Coverage Status](https://coveralls.io/repos/tomwalder/php-gds/badge.svg?branch=2.0)](https://coveralls.io/r/tomwalder/php-gds)

# Google Datastore Library for PHP #

[Google Cloud Datastore](https://cloud.google.com/datastore/) is a great NoSQL solution (hosted, scalable, free up to a point), but it can be tricky (i.e. there's lots of code glue needed) to get even the "Hello World" of data persistence up and running in PHP.

This library is intended to make it easier for you to get started with and to use Datastore in your applications.

## Version 2.x ALPHA ##

**Please note, this 2.0-series branch is in ALPHA and not suitable (yet) for production**

The documentation is not yet fully representative of 2.x implementation.

## Table of Contents ##

- [Examples](#examples)
- [New in 2.0](#new-in-version-20)
- [Getting Started](#getting-started)
- [Defining Your Model](#defining-your-model)
- [Creating Records](#creating-records)
- [Queries, GQL & The Default Query](#queries-gql--the-default-query)
- [Multi-tenant Applications & Data Namespaces](#multi-tenant-applications--data-namespaces)
- [Entity Groups, Hierarchy & Ancestors](#entity-groups-hierarchy--ancestors)
- [Transactions](#transactions)
- [More About Google Cloud Datastore](#more-about-google-cloud-datastore)
- [Unit Tests](#unit-tests)
- [Footnotes](#footnotes)

## Examples ##

I find examples a great way to decide if I want to even try out a library, so here's a couple for you. 

```php
// Build a new entity
$obj_book = new GDS\Entity();
$obj_book->title = 'Romeo and Juliet';
$obj_book->author = 'William Shakespeare';
$obj_book->isbn = '1840224339';

// Write it to Datastore
$obj_store = new GDS\Store('Book');
$obj_store->upsert($obj_book);
```

See below for [Alternative Array Syntax](#alternative-array-syntax) for creating Entities.

Now let's fetch all the Books from the Datastore and display their titles and ISBN numbers

```php
$obj_store = new GDS\Store('Book');
foreach($obj_store->fetchAll() as $obj_book) {
    echo "Title: {$obj_book->title}, ISBN: {$obj_book->isbn} <br />", PHP_EOL;
}
```

### More Examples ###

These initial examples assume you are either running a Google AppEngine application or in a local AppEngine dev environment. 
In both of these cases, we can auto detect the **dataset** and use the default ***Protocol Buffer Gateway*** (new in 2.0).

We use a `GDS\Store` to read and write `GDS\Entity` objects to and from Datastore. 

These examples use the generic `GDS\Entity` class with a dynamic Schema. See [Defining Your Model](#defining-your-model) below for more details on custom Schemas and indexed fields.

Check out the [examples](examples/) folder for many more and fuller code samples.

### Using the Google PHP API and the JSON Datastore API ###

A little more configuration is required if you want or need to use the JSON API instead of Protocol Buffers.

The Store needs a `GDS\Gateway` to talk to Google and the gateway needs a `Google_Client` for authentication.

```php
$obj_client = GDS\Gateway\GoogleAPIClient::createGoogleClient(APP_NAME, ACCOUNT_NAME, KEY_FILE);
$obj_gateway = new GDS\Gateway\GoogleAPIClient($obj_client, DATASET_ID);
$obj_book_store = new GDS\Store('Book', $obj_gateway);
```

### Demo Application ###

A simple guest book application

Application: http://php-gds-demo.appspot.com/

Code: https://github.com/tomwalder/php-gds-demo

## New in Version 2.0 ##

New features in 2.0 include 
* **Faster!** Google Protocol Buffer allows faster, low-level access to Datastore
* **Easier to use** - sensible defaults and auto-detection for AppEngine environments
* **Less dependencies** - no need for the Google PHP API Client, unless running remote or from non-AppEngine environments
* Suite of unit tests
* Optional drop-in JSON API Gateway for remote or non-AppEngine environments (this was the only Gateway in 1.x)

### Backwards Compatibility ###

The library is *almost* fully backwards compatible. And in fact, the main operations of the `GDS\Store` class are identical. 

There is one BC-break in 2.0 - the re-ordering of construction parameters for the `GDS\Store` class.

`GDS\Store::__construct(<Kind or Schema>, <Gateway>)`

instead of 

`GDS\Store::__construct(<Gateway>, <Kind or Schema>)`

This is because the Gateway is now optional, an has a sensible, automated, default - the new Protocol Buffer implementation.

## Getting Started ##

Are you sitting comfortably? before we begin, you will need: 
- a Google Account (doh), usually for running AppEngine - but not always
- a Project to work on with the "Google Cloud Datastore API" turned ON [Google Developer Console](https://console.developers.google.com/)

If you want to use the JSON API from remote or non-AppEngine environments, you will also need
- a "Service account" and a P12 key file for that service account [Service Accounts](https://developers.google.com/accounts/docs/OAuth2#serviceaccount)

### Composer, Dependencies ###

To install using composer, use this require line

`"tomwalder/php-gds": "2.0.*@dev"`

## Defining Your Model ##

Because Datastore is schemaless, the library also supports fields/properties that are not explicitly defined. But it often makes a lot of sense to define your Entity Schema up front.

Here is how we might build the Schema for our examples, with a Datastore Entity Kind of "Book" and 3 fields.

```php
$obj_schema = (new GDS\Schema('Book'))
   ->addString('title')
   ->addString('author')
   ->addString('isbn');
   
// The Store accepts a Schema object or Kind name as it's first parameter
$obj_book_store = new GDS\Store($obj_schema);
```

By default, all fields are indexed. An indexed field can be used in a WHERE clause. You can explicitly configure a field to be not indexed by passing in `FALSE` as the second parameter to `addString()`.

If you use a dynamic schema (i.e. do not define on, but just use the Entity name) then all fields will be indexed for that record.

Avaialable Schema configuration methods:
- `GDS\Schema::addString`
- `GDS\Schema::addInteger`
- `GDS\Schema::addDatetime`
- `GDS\Schema::addFloat`
- `GDS\Schema::addBoolean`
- `GDS\Schema::addStringList`

Take a look at the `examples` folder for a fully operational set of code.

## Creating Records ##

### Alternative Array Syntax ###

There is an alternative to directly constructing a new `GDS\Entity` and setting it's member data, which is to use the `GDS\Store::createEntity` factory method as follows.

```php
$obj_book = $obj_book_store->createEntity([
    'title' => 'The Merchant of Venice',
    'author' => 'William Shakespeare',
    'isbn' => '1840224312'
]);
```

Support for DateTime object binding was added recently (also see query parameter binding below)

```php
$obj_book = $obj_book_store->createEntity([
    'title' => 'Some Book',
    'author' => 'A N Other Guy',
    'isbn' => '1840224313',
    'published' => new DateTime('-5 years')
]);
```

## Queries, GQL & The Default Query ##

At the time of writing, the `GDS\Store` object uses Datastore GQL as it's query language. Here is an example:

```php
$obj_book_store->fetchOne("SELECT * FROM Book WHERE isbn = '1853260304'");
```

And with support for named parameter binding (strings, integers)

 ```php
$obj_book_store->fetchOne("SELECT * FROM Book WHERE isbn = @isbnNumber", [
    'isbnNumber' => '1853260304'
]);
```

Support for DateTime object binding

 ```php
$obj_book_store->fetchOne("SELECT * FROM Task WHERE date_date < @now", [
    'now' => new DateTime()
]);
```

We provide a couple of helper methods for some common (root Entity) queries, single and batch  (much more efficient than many individual fetch calls):

- `GDS\Store::fetchById`
- `GDS\Store::fetchByIds` - batch fetching
- `GDS\Store::fetchByName`
- `GDS\Store::fetchByNames` - batch fetching

When you instantiate a store object, like `BookStore` in our example, it comes pre-loaded with a default GQL query of the following form (this is "The Default Query")

```sql
SELECT * FROM <Kind> ORDER BY __key__ ASC
```

Which means you can quickly and easily get one or many records without needing to write any GQL, like this:

```php
$obj_store->fetchOne();     // Gets the first book
$obj_store->fetchAll();     // Gets all books
$obj_store->fetchPage(10);  // Gets the first 10 books
```

### Pagination ###

When working with larger data sets, it can be useful to page through results in smaller batches. Here's an example paging through all Books in 50's.

```php
$obj_book_store->query('SELECT * FROM Book');
while($arr_page = $obj_book_store->fetchPage(50)) {
    echo "Page contains ", count($arr_page), " records", PHP_EOL;
}
```

#### Limits, Offsets & Cursors ####

In a standard SQL environment, the above pagination would look something like this:

- `SELECT * FROM Book LIMIT 0, 50` for the first page 
- `SELECT * FROM Book LIMIT 50, 50` for the second, and so on.

Although you can use a very similar syntax with Datastore GQL, it can be unnecessarily costly. This is because each row scanned when running a query is charged for. So, doing the equivalent of `LIMIT 5000, 50` will count as 5,050 reads - not just the 50 we actually get back.

This is all fixed by using Cursors. The implementation is all encapsulated within the `GDS\Gateway` class so you don't need to worry about it.

Bototm line: the bult-in pagination uses Cursors whenever possible for fastest & cheapest results.

#### Tips for LIMIT-ed fetch operations ####

Do not supply a `LIMIT` clause when calling 
- `GDS\Store::fetchOne` - it's done for you (we add `LIMIT 1`)
- `GDS\Store::fetchPage` - again, it's done for you and it will cause a conflict. 

#### Pricing & Cursor References ####

- [Query Cursors](https://cloud.google.com/datastore/docs/concepts/queries#Datastore_Query_cursors)
- [Costs for Datastore Calls](https://cloud.google.com/appengine/pricing)
- [Datastore Quotas](https://cloud.google.com/appengine/docs/quotas#Datastore)

## Multi-tenant Applications & Data Namespaces ##

Google Datastore supports segregating data within a single "Dataset" using something called Namespaces.

Generally, this is intended for multi-tenant applications where each customer would have separate data, even within the same "Kind".

This library supports namespaces, and they are be configured per `Gateway` instance by passing in the optional 3rd namespace parameter.

<strong>ALL operations carried out through a Gateway with a namespace configured are done in the context of that namespace. The namespace is automatically applied to Keys when doing upsert/delete/fetch-by-key and to Requests when running GQL queries.</strong>

```php
// Create a store for a particular customer or 'application namespace'
$obj_client = GDS\Gateway::createGoogleClient(APP_NAME, ACCOUNT_NAME, KEY_FILE);
$obj_namespaced_gateway = new GDS\Gateway($obj_client, DATASET_ID, 'customer-namespace');
$obj_namespaced_book_store = new BookStore($obj_namespaced_gateway);
```

Further examples are included in the examples folder.

## Entity Groups, Hierarchy & Ancestors ##

Google Datastore allows for (and encourages) Entities to be organised in a hierarchy.

The hierarchy allows for some amount of "relational" data. e.g. a `ForumThread` entity might have one more more `ForumPosts` entities as children.

Entity groups are quite an advanced topic, but can positively affect your application in a number of areas including
 
- Transactional integrity
- Strongly consistent data

At the time of writing, I support working with entity groups through the following methods

- `GDS\Entity::setAncestry`
- `GDS\Entity::getAncestry`
- `GDS\Store::fetchEntityGroup`

## Transactions ##

The `GDS\Store` supports running updates and deletes in transactions.

To start a transaction

```php
$obj_store->beginTransaction();
```

Then, any operation that changes data will commit *and consume* the transaction. So an immediate call to another operation WILL NOT BE TRANSACTIONAL.

```php
// Data changed within a transaction
$obj_store->upsert($obj_entity);

// Not transactional
$obj_store->delete($obj_entity);
```

## Custom Entities and Stores ##

> @todo v2.0 revisions

Whilst you can use the `GDS\Entity` and `GDS\Store` classes directly, as per the examples above, you may find it useful to extend both and have the extended Store contain the Schema definition.

For example

```php
class Book extends GDS\Entity { /* ... */ }
class BookStore extends GDS\Store { /* ... */ }
```

This way, when you pull objects out of Datastore, they are objects of your defined Entity class.

```php
$obj_store = new BookStore($obj_gateway);
$obj_book = $obj_store->fetchOne(); // $obj_book will be a "Book" object
```

Check out the examples folder for `Book.php` and `BookStore.php` code samples.

## Re-indexing ##

When you change a field from non-indexed to indexed you will need to "re-index" all your existing entities before they will be returned in queries run against that index by Datastore. This is due to the way Google update their BigTable indexes.

I've included a simple example (paginated) re-index script in the examples folder, `reindex.php`.

## More About Google Cloud Datastore ##

What Google say:

> "Use a managed, NoSQL, schemaless database for storing non-relational data. Cloud Datastore automatically scales as you need it and supports transactions as well as robust, SQL-like queries."

https://cloud.google.com/datastore/

### Specific Topics ###

A few highlighted topics you might want to read up on
- [Entities, Data Types etc.](https://cloud.google.com/datastore/docs/concepts/entities)
- [More information on GQL](https://cloud.google.com/datastore/docs/concepts/gql)
- [GQL Reference](https://cloud.google.com/datastore/docs/apis/gql/gql_reference)
- [Indexes](https://cloud.google.com/datastore/docs/concepts/indexes)
- [Ancestors](https://cloud.google.com/datastore/docs/concepts/entities#Datastore_Ancestor_paths)
- [More about Datastore Transactions](https://cloud.google.com/datastore/docs/concepts/transactions)

## Unit Tests ##

A full suite of unit tests is in the works. [Click here for more details](tests/).

## Footnotes ##

I am certainly more familiar with SQL and relational data models so I think that may end up coming across in the code - rightly so or not!

