# Lounge

Simple Mongoose-inspired ODM for [Couchbase](http://www.couchbase.com).

## Installation

`npm install lounge`

## Stability

This module is under development and there could be bugs. API may not be 100% locked down. 
Documentation is still work in progress.

## Overview

Lounge is a simple, somewhat opinionated, Mongoose-inspired ODM for [Couchbase](http://www.couchbase.com). Main goal is
to provide modeling tool framework for working with Couchbase databases in an asynchronous environment of Node.js. 

```js
var lounge = require('lounge');
lounge.connect('couchbase://127.0.0.1');

var schema = lounge.schema({ name: String });
var Cat = lounge.model('Cat', schema);

var kitty = new Cat({ name: 'Zildjian' });
kitty.save(function (err) {
  if (err) // ...
  console.log('meow');
});
```

##### Features:

* Schema definition
* Strict modelling based on schema
* Schema extension
* Automatic type validation
* Document upsert and removal
* Embedded (referenced) documents
* Automatic and manual population of embedded (referenced) document
* Middleware including pre and post hooks
* Indexing using [reference lookup documents](http://docs.couchbase.com/developer/dev-guide-3.0/lookups.html)

##### Outside of the scope of this module:

* Document and view management. There are two many patterns and ways of performing document and view management and 
 view lookup that it is impractical to accomplish anything sane within a simple ODM. This can easily be expanded
 on top of Lounge.
* View queries. For same reasons this falls outside of the scope of Lounge.
* Automatic document removal on key change. That is if a document key property changes, the new document is saved under
the new key. The old document sticks around under the old key. There are too many implications if we start automatically
handle document removal in this scenario. This should be handled by the user of this module.

## Documentation

* [Setup](#lounge)
* [Modelling](#model)
* [Middleware](#middleware)
* [Schema Extension](#schema-extend)
* [Embedded Documents](#embedded)
* [Saving Documents](#saving)
* [Removing Documents](#removing)
* [Population](#population)
* [Indexes](#indexes)
* [Queries](#queries)
* [Events](#events)

### Setup <a id="lounge"></a>

```js
var lounge = require('lounge');
```

Module exports an instance of `Lounge` class.

#### Lounge(options)

Creates a new instance of `Lounge` class. You rarely have to do this.

```js
var lounge = require('lounge');
var l = new lounge.Lounge();
```

**Options**

* `keyPrefix` - key prefix for all keys. No default. Generally useful if you wish to namespace documents. 
Example: `app::env::`.
* `storeFullReferenceId` - whether to store embedded document keys as fully expanded keys 
(with prefix and suffix applied) or just the minimized version. default: `false`. 
* `storeFullKey` - Similarly, to store the fully expanded document key inside the key property. default: `false`
* `alwaysReturnArrays` - set to true to force `findyById` to always return an array of documents even if only
a single key is passed in. Default: `false`
* `refIndexKeyPrefix` - reference lookup index document key prefix. The name of the index is appended to this.
Default: `'$_ref_by_'`
* `delimiter` - delimiter string used for concatenation in reference document key expansion / generation. default: `'_'` 
This is prepended to the reference document key.
* `waitForIndex` - When documents are saved, indexes are updated. We can wait for this operation to finish before 
returning from `save()`. Default: `false`
* `minimize` - "minimize" schemas by removing empty objects. Default: `true`

Any of these options, or additional variables can be manipulated using the `setOption` and `getOption` methods.
  
```js
var lounge = require('lounge');
lounge.setOption('alwaysReturnArrays', true);
console.log(lounge.getOption('alwaysReturnArrays');
```

#### Lounge.connect(options, fn, mock)

Connects to the database cluster based on `options`. When completed calls `fn` callback. Set `mock` to `true` to use
[Couchbase mocking](https://github.com/couchbase/couchnode#mock-testing). Alternatively you can set
`LOUNGE_COUCHBASE_MOCK` environment variable.

**Options**

* `connectionString` - connection string for the cluster. See [Cluster documentation](http://docs.couchbase.com/sdk-api/couchbase-node-client-2.1.2/Cluster.html).
* `bucket` - name of the bucket to be used for [`openBucket`](http://docs.couchbase.com/sdk-api/couchbase-node-client-2.1.2/Cluster.html#openBucket)
or the actual, already connected, Couchbase `bucket` instance.
* `password` - password for [`openBucket`](http://docs.couchbase.com/sdk-api/couchbase-node-client-2.1.2/Cluster.html#openBucket)
* `certpath` - certpath for [cluster constructor](http://docs.couchbase.com/sdk-api/couchbase-node-client-2.1.2/Cluster.html).

```js
var lounge = require('lounge');
lounge.connect('couchbase://127.0.0.1', function(err) {
  // ... connected
});
```

#### Lounge.disconnect()

Disconnect from the bucket. Deletes all defined models.

#### Lounge.schema(descriptor, options)

Creates a new `Schema` based on `descriptor` and `options`. Prefer this over the actual `Schema` constructor as this
will pass lounge config variables to the `Schema` constructor automatically. 

Schema construction options:

* `keyPrefix` - key prefix for all keys. No default. Generally useful if you wish to namespace documents. Example: `app::env::`.
* `keySuffix` - Similar as prefix but used as a suffix
* `refIndexKeyPrefix` - reference lookup index document key prefix. The name of the index is appended. Default: '$_ref_by_'
* `delimiter` - delimiter string used for concatenation in reference document key expansion / generation.
Default: '_'. This is prepended to the reference document key.
*`minimize` - "minimize" schemas by removing empty objects. Default: `true`
* `toObject` - toObject method options: `transform`, `virtuals` and `minimize`
* `toJSON` - toJSON method options, similar to above
* `strict` - ensures that value passed in ot assigned that were not specified in our schema do not get saved. Default: `true`


#### Lounge.model(name, schema, options)

Defines a Model based on case sensitive model `name` and created `schema`, with given options. From there we can create
instances of the Model;

**Options**

* `freeze` - to freeze the model. See `Object.freeze`. Default: `true`

```js
var userSchema = lounge.schema({
  firstName: String,
  lastName: String,
  email: String,
  dateOfBirth: Date,
});

User = lounge.model('User', userSchema);
var user = new User({
  firstName: 'Joe',
  lastName: 'Smith',
  email: 'joe@gmail.com',
  dateOfBirth: new Date()
});

console.log(user instanceof User) // true
```

#### Lounge.modelNames()

Returns and array of all defined model names.

#### Bucket functions

Lounge instance also inherits all public [`Bucket`](http://docs.couchbase.com/sdk-api/couchbase-node-client-2.1.2/Bucket.html)
 functions and properties.
 
```js
lounge.get('mydocumentkey', function(err, res) {
  // .. save as if we did normal bucket.get()
});
```

#### Lounge.Schema

Exported `Schema` constructor. You can use this as well as `lounge.schema()`, which is preferred.

```js
var userSchema = new lounge.Schema({ name: String });
```

#### Lounge.Model

Exported `Model` constructor. You should never have to manually create a Model.

#### Lounge.Document

Exported  internal `Document` constructor. You should never have to manually create a Document.
 
```js
// already defined User model
var user = new User({name: 'Bob'});
console.log(user instanceof User) // true
console.log(user instanceof lounge.Model) // true
console.log(user instanceof lounge.Document) // true
```

### Modelling <a id="model"></a>

**Basics**

We begin defining a data model using a schema.

```js
var userSchema = lounge.schema({
  firstName: String,
  lastName: String,
  age: Number,
  usernames: [String],
  setup: Boolean
  metadata: {
    createdAt: Date,
    updatedAt: Date
  }
});
```

We can add additional properties using `add` function:

```js
userSchema.add('name', String);
```

Alternatively we can explicitly specify the type using `type` property:

```js
var catSchema = lounge.schema({
  name: { type: String }
  breed: String,
});

catSchema.add('age', {type: String});
```

Schema options can be set at construction or using the `set` function.
 
```js
var catSchema = lounge.schema({
  name: { type: String }
  breed: String,
}, {
  keyPrefix: 'cat::'
});

catSchema.set('refIndexKeyPrefix', '::');
```

**Document keys**

By defualt schemas come with an `id` property as the document key, and the automatically generated value will be
a [UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier) 
using [node-uuid](https://www.npmjs.com/package/node-uuid) `v4()` function. This should be most practical and most
appropriate in a lot of cases. Alternatively you can specify explicit key properties:

```js
var userSchema = lounge.schema({
  firstName: String,
  lastName: String,
  email: { type: String, key: true, generate: false }
});
```

Here we desire `email` to be used as the document key and we specify `generate: false` because we do not want Lounge
to automatically handle key property value generation. If we still want uuid generation but in a different property 
we can specify so:

```js
var userSchema = lounge.schema({
  firstName: String,
  lastName: String,
  email: String,
  userId: {type: String, key: true, generate: true }
});
```

`generate` does not have to be set explicitly to `true` as that is the default.
 
We can specify additional prefix and/or suffix for keys. This will be used when wrigin to the database as the actual
document key.

```js
var userSchema = lounge.schema({
  firstName: String,
  lastName: String,
  email: { type: String, key: true, generate: false, prefix: 'user'}
});
```

Note that setting prefix and suffix options like this will take presidence over any `keyPrefix` and `keySuffix` 
options specified in the second options parameter to the `schema()` call or any settings in the lounge config.

**Examples**

```js
var lounge = require('lounge');

// ... connect

var userSchema = lounge.schema({
  name: String
  email: { type: String, key: true, generate: false, prefix: 'user::'}
});

var User = lounge.model('User', userSchema);
var user = new User({name: 'Bob Smith', email: 'bsmith@acme.com'});
user.save();
```

This will save the user document under key `user::bsmith@acme.com`.

```js
var lounge = require('lounge');

// ... connect

var userSchema = lounge.schema({
  name: String
}, {
  keyPrefix: 'user::'
});

var User = lounge.model('User', userSchema);
var user = new User({name: 'Bob Smith'});
user.save();
```

This will automatically generate a uuid `id` property and save the user document under key 
similar to `user::110ec58a-a0f2-4ac4-8393-c866d813b8d1`.

**Data manipulation**

Data in Model instances can be access directly or using `get` function. Similarly it can be manipulated using 
either assignment operator or using the `set` function. In either case the input value is validated to be of proper type.
 
```js
var userSchema = lounge.schema({
  name: String
  friends: Number,
  dob: Date,
  setup: Boolean
});

var User = lounge.model('User', userSchema);
var user = new User({name: 'Bob Smith'});

user.get('name'); // 'Bob Smith'
user.name = 'Joe'; // OK
console.log(user.name); // 'Joe'
user.set('friends', 20); // OK
user.friends = 'abc'; // nope. still 20
user.dob = new Date('July 5, 1980');
user.get('dob'); // instance of Date
user.set('setup', 'yup'); // nope
user.setup = true; // OK
```

**Virtuals**

Virtuals are document properties that you can get and set but that do not get persisted to teh database. 
The getters are useful for formatting or combining fields, while setters are useful for de-composing a single value 
into multiple values for storage.

```js
var userSchema = lounge.schema({
  firstName: String, 
  lastName: String
});

userSchema.virtual('fullName', {
  get: function () {
    return this.firstName + ' ' + this.lastName;
  },
  set: function (v) {
    if (v !== undefined) {
      var parts = v.split(' ');
      this.firstName = parts[0];
      this.lastName = parts[1];
    }
  }
});

var User = lounge.model('User', userSchema);
var user = new User({firstName: 'Bob', lastName: 'Smith'});
console.log(user.firstName); // Bob Smith
```

If no `set` function is defined the virtual is read-only.

**Statics**

Adding static methods to Models can be accomplished using `static()` schema function

```js
var userSchema = lounge.schema({
  firstName: String, 
  lastName: String
});

userSchema.static('foo', function(p, q) {
  return p + q;
});

var User = lounge.model('User', userSchema);
User.foo(1, 2); // 3
```

We can also pass an object of function keys and function values, and they will all be added.

**Methods**

Similarly adding instance methods to Models can be done using `method()` schema function. 

```js
var userSchema = lounge.schema({
  firstName: String, 
  lastName: String
});

userSchema.method('fullName', function() {
  return this.firstName + ' ' + this.lastName;
});

var User = lounge.model('User', userSchema);
var user = new User({firstName: 'Bob', lastName: 'Smith'});
user.fullName(); // 'Bob Smith'
```

We can also pass an object of function keys and function values, and they will all be added.

**init() method**

There is a special `init` method that if specified in schema definition will be called at the end of model creation. 
You can do additional setup here. This method is not passed in any arguments.

**$_data variable**

Since by default models are created using `strict` mode, Model instances setup and expose an internal `$_data` variable 
that can be used for any internal storage that's not part of model definition as specified inside of schema. 
This variable is not saved into the database. This can be used in combination with `init` function:

```js
var userSchema = lounge.schema({
  firstName: String,
  lastName: String,
  email: String
});

userSchema.method('init', function() {
  // store any logic stuff into $_data
  this.$_data.initialEmal = this.email;
  // later we can see if email was changed for example
});
```

### Middleware <a id="middleware"></a>

Similar to Mongoose middleware, Lounge exposes `pre` and `post` [hooks](https://www.npmjs.com/package/hooks-fixed).
Normally this is used to do additional validation pre save, and cleanup post document removal. Although the hooks can be
setup for any method including `toObject` and `toJON` methods.


```js
var userSchema = lounge.schema({
  firstName: String,
  lastName: String,
  email: String
});

userSchema.pre('save', function (next) {
  if (this.email) {
    this.email = this.email.toLowerCase();
  }

  // we must call next to continue control flow 
  next();
});

userSchema.post('remove', function () {
  console.log('document %s removed', this.id);
});
```

Important note here is that post 'save' and 'remove' hooks do not receive any form of control flow. There are no
callbacks passed.

The callback passed into pre hooks can be used to control flow of logic:

```js
schema.pre('save', function (next) {
  // some custom validation method
  if (!this.validate()) {
    return next(new Error('Validation error!'));
  }
 
  next();
});

// elsewhere...

doc.save(function(err, savedDoc) {
  if(err) console.log(err); // 'Validation error!' Document was not saved
});
```

### Schema Extension <a id="schema-extend"></a>

It is useful to have a common base schema, that all other schemas / models would extend or "inherit" properties from.
This can be accomplished by either using the `Schema.extend` function. When performed all property definitions, virtuals,
methods, statics, and middleware that are present in the base schema **but not** present in destination schema are copied 
into the destination schema.

```js
 var baseSchema = lounge.schema({
  metadata: {
    doc_type: String,
    createdAt: Date,
    updatedAt: Date
  }
});

baseSchema.pre('save', function (next) {
  if (!this.metadata) {
    this.metadata = {};
  }

  var now = new Date();

  if (!this.metadata.createdAt) {
    this.metadata.createdAt = now;
  }

  this.metadata.updatedAt = now;
  this.metadata.doc_type = this.modelName;

  next();
});

baseSchema.method('baseFoo', function () {
  console.log('base foo');
});

var userSchema = lounge.schema({
  name: String,
  email: String,
});

userSchema.pre('save', function (next) {
  if (this.email) {
    this.email = this.email.toLowerCase();
  }

  next();
});

userSchema.method('userFoo', function () {
  console.log('user foo');
});

userSchema.extend(baseSchema);
var User = lounge.model('User', userSchema);

user = new User({
  name: 'Bob Smith',
  email: 'BSmith@gmail.com'
});

user.baseFoo() // prints 'base foo'
user.userFoo() // prints 'user foo'

user.save(function(err, savedDoc) {
  console.log(user.metadata.updatedAt); // Sat Dec 29 2015 03:30:00 GMT-0400 (AST)
  console.log(user.metadata.doc_type); // 'user'
  console.log(user.email); // 'bsmith@gmail.com'
});
```

### Embedded Documents <a id="embedded"></a>

Lounge allows for embedding and referencing other Models within schema.

```js
var addressSchema = lounge.schema({
  street: String,
  city: String,
  country: String
});

var Address = lounge.model('Address', addressSchema);

var blogPostSchema = lounge.schema({
  title: String,
  body: String,
});

var BlogPost = lounge.model('BlogPost', blogPostSchema);

var userSchema = lounge.schema({
  name: String,
  address: Address,
  posts: [BlogPost]
});

var post = new BlogPost({
  title: 'Foo',
  body: 'Lorem ipsum'
});

var user = new User({
  name: 'Bob Smith',
  posts: [post],
  address: new Address({
    street: '123 Fake Street',
    city: 'Springfield',
    country: 'USA'
  })
});
```

You can manipulate and work with subdocument just like any model instances. When the top level document is saved
all child sub-documents are saved as well. Subdocuments **must** be an instance of the Model defined in the schema or a 
`String` in which case it represents the key / id of the subdocument.

### Saving Documents <a id="saving"></a>

Go deeper into saving documents.
 
### Getting Documents <a id="getting"></a>

Go deeper into getting documents.

### Population <a id="population"></a>

Go deeper into population

### Removing Documents <a id="removing"></a>

Go deeper into removing documents.

### Indexes <a id="indexes"></a>

Go deeper into indexes

### Index queries <a id="queries"></a>

Go deeper into indexes

### Events <a id="events"></a>

Go deeper into events

## TODO

See [TODO.md](https://github.com/bojand/lounge/blob/v1/TODO.md).

## Credits

Credits for draft, mongoose, other things.

## License

Copyright 2015 Bojan Djurkovic

Licensed under the MIT License.