mongodb-schema
==============

Infer probabilistic schema of javascript objects or a MongoDB collection. 

This package is dual-purpose. It serves as a [node.js module](#usage-with-nodejs) and can also be used in the [MongoDB](#usage-with-mongodb) shell directly, where it extends the `DBCollection` shell object.

_mongodb-schema_ is an early prototype. Use at your own risk.

<br>

## Usage with Node.js

### Installation
Install the script with:

```
npm install mongodb-schema
```

### Usage 

Load the module and use call `schema( documents, options, callback )`, which will call `callback(err, res)` with an error or the result once it's done analysing the documents.

```js
var schema = require('mongodb-schema');

// define some documents
var documents = [
    {a: 1},
    {a: {b: "hello"}}
];

// call with options and callback function
schema( documents, {flat: true}, function (err, res) {
    if (err) return console.error( err );
    console.log( JSON.stringify( res, null, "\t" ) );
})
```

This would output:
```json
{
    "$count": 2,
    "a": {
        "$count": 2,
        "$type": {
            "number": 1,
            "object": 1
        },
        "$prob": 1
    },
    "a.b": {
        "$count": 1,
        "$type": "string",
        "$prob": 0.5
    }
}
```

<br>

## Usage with MongoDB

### Installation

There are two ways to load the script, one-time (if you just want to test how it works) and permanent (for frequent use).

#### 1. Load the script directly (one-time usage)

Call the `mongo` shell this command: 

```
mongo <basepath>/lib/mongodb-schema.js --shell
```

It will first load `mongodb-schema.js` and the open the shell as usual. You will have to add the script every time you open the shell. Replace the `<basepath>` part with the actual path where the `mongodb-schema` folder is located.

#### 2. Load the script via the `.mongorc.js` file (permanent usage)

You can also add the following line to your `~/.mongorc.js` file to always load the file on shell startup (unless started with `--norc`):

```js
load('<basepath>/lib/mongodb-schema.js')
```

Replace the `<basepath>` part with the actual path where the `mongodb-schema` folder is located.


### Usage

##### Basic Usage

The script extends the `DBCollection` object to have another new method: `.schema()`. On a collection called `foo`, run it with:

```js
db.foo.schema()
```

This will use the first 100 (by default) documents from the collection and calculate a probabilistic schema based on these documents.

##### Usage with options

You can pass in an options object into the `.schema()` method. See [Options]() below. Example:

```js
db.foo.schema( {samples: 20, flat: true} )
```

This will use the first 20 documents to calculate the schema and return the schema as flat object (all fields are collapsed to the top with dot-notation). See the [Examples](#examples) section below for nested vs. flat schemata. 

<br>

## Schema Format

The schema format is in JSON, and the shape resembles the shape of a superposition of all inferred documents. Each level of the schema (from the root level down into each nested sub-document) has annotations with special meta-variables, which by default start with a `$`. Examples of such annotations are `$count`, `$type`, etc.

### Example

Here's a first example, created from a set of 3 documents: 
```js

schema([ 
  {"a": 1, "b": "foo", "c": {"d": null, "e": 4.3}},
  {"a": 2, "b": "bar"},
  {"a": 3, "b": "baz", "c": {"d": "boo", "e": 2.9}}
])

// output
{
    "$count": 3,
    "a": {
        "$count": 3,
        "$type": "number",
        "$prob": 1
    },
    "b": {
        "$count": 3,
        "$type": "string",
        "$prob": 1
    },
    "c": {
        "$count": 2,
        "d": {
            "$count": 2,
            "$type": {
                "null": 1,
                "string": 1
            },
            "$prob": 1
        },
        "e": {
            "$count": 2,
            "$type": "number",
            "$prob": 1
        },
        "$prob": 0.6666666666666666
    }
}
```

A lot going on here already. There is a top-level `$count`, that just counts all the parsed documents. Each of the first-level sub-documents `"a"`, `"b"`, `"c"` get their own section in the schema, just as if all documents were superimposed on top of each other (think "transparent slides"). Each sub-document has a `$count` of its own, together with `$type` information and a probability `$prob`. These fields are explained below. 

### Sampling Size

The MongoDB shell version of the script has an additional parameter `samples`, which by default is set to 100 and limits the number of samples to 100. You can change it to another value, or use the option `{samples: 'all'}` to look at all the documents (careful: this can be computationally expensive, depending on the number of documents).

This option is not available when used as a stand-alone javascript or node module.


### Flat Format

If you pass in the option `{flat: true}` as second parameter to `schema`, every sub-level is flattened down to the root level, using dot-notation. Here is the same schema as above, but with the flat option:

```js
{
    "$count": 3,
    "a": {
        "$count": 3,
        "$type": "number",
        "$prob": 1
    },
    "b": {
        "$count": 3,
        "$type": "string",
        "$prob": 1
    },
    "c": {
        "$count": 2,
        "$prob": 0.6666666666666666
    },
    "c.d": {
        "$count": 2,
        "$type": {
            "null": 1,
            "string": 1
        },
        "$prob": 1
    },
    "c.e": {
        "$count": 2,
        "$type": "number",
        "$prob": 1
    }
}
```

Notice how the `"c"` sub-document doesn't contain information about its sub fields `"d"` and `"e"`. This is now tracked separate under `"c.d"` and `"d.e"`.

### Data Inference

You can enable data inference mode with the `{data: true}` option. The schema analyser will then gather statistics of your data for each field. The kind of information that is collected depends on the data type.


##### Numbers and Dates

For numbers and dates, you will get  some statistics under the `$stats`  field, with `min` and `max` value of all the documents seen. Example:
```js
schema([ 
    {"a": 2}, {"a": 8}, {"a": 1}, {"a": 7}
], {data: true})

// output
{
    "$count": 4,
    "a": {
        "$count": 4,
        "$type": "number",
        "$stats": {
            "min": 1,
            "max": 8
        },
        "$prob": 1
    }
}
```

#### Categories

When you enable data inference, all sampled values are stored, and if there are some duplicate values, the `$category` flag is set to `true` on the field and the histogram of values is returned under `$hist`. The shape of histogram values is `{v: <value>, c: <count>}`, for example: 

```
$hist: [
    {v: "foo", c: 62},
    {v: "bar", c: 38}
]
```

The choice for array of documents with `v` and `c` keys was made because non-string values, like dates and numbers, cannot be JSON keys. 

The maximum cardinality is set to 100. If there are more categories, an additional array element `{o: <count>}` is included where `o` stands for "other". This is to limit the amount of memory needed to keep the histogram stats. The maximum cardinality can be configured with the `data.maxCardinality` value. Instead of assigning `true` to the data option, you can pass in a sub-document to set the maximum cardinality:

Example:

```js
schema([
    {a: "a"}, {a: "a"}, {a: "b"}, {a: "c"}, {a: "d"}, {a: "e"}, {a: "f"}
], { data: { maxCardinality: 3 }});

// output
{
    "$count": 7,
    "a": {
        "$count": 7,
        "$type": "string",
        "$category": true,
        "$hist": [
            {"v": "a", "c": 2},
            {"v": "b", "c": 1},
            {"v": "c", "c": 1},
            {"o": 3}
        ],
        "$prob": 1
    }
}
```


#### Counts and Probabilities

The schema keeps count of the number of documents and sub-documents on each level. This information is stored in the `$count` field. If we pass in a single empty document `{}`, the output is this: 

```json
{   
    "$count": 1,
}
```

Passing in a document with a field `{a: 1}` returns this schema:

```json
{   
    "$count": 1,
    "a": {
        "$count": 1,
        "$type": "number",
        "$prob": 1
    }
}
```

The `"a"` sub-document receives its own `$count` field and only counts the number of occurences where the the `"a"` sub-document was present. Another example, for this list of documents: `[ {a: 1}, {b: 1}, {a: 0}, {a: 2}, {b: 5} ]`:

```json
{
    "$count": 5,
    "a": {
        "$count": 3,
        "$type": "number",
        "$prob": 0.6
    },
    "b": {
        "$count": 2,
        "$type": "number",
        "$prob": 0.4
    }
}
```

We can see a total of 5 documents (top-level `"$count"`) and `"a"` was present 3 times, `"b"` twice. 
<br>

Additionally, the schema contains a `$prob` value, indicating the relative probability for a sub-document given its parent document. 

#### Array Collapsing

Arrays are not handled as a distinct data type. Instead, they are collapsed and interpreted as individual values for the given field. This is similar to how MongoDB treats arrays in context of querying: `db.coll.find({a: 1})` will match documents like `{a: [1, 4, 9]}`.

Because each value of the array is treated as a separate instance of the sub-document, this affects the statistics like `$count` and `$prob`, and you can end up with a probability larger than 1, as it represents the average length of the array. 

To indentify a schema with a collapsed array field, the `$array` flag is set to `true` if at least one array was collapsed for a given field.

Example:

```js
schema([
    {a: [1, 2, 3, 4]}, 
    {a: [5, 6]}
])

// output
{
    "$count": 2,
    "a": {
        "$count": 6,
        "$type": "number",
        "$array": true,
        "$prob": 3
    }
}
```


#### Meta Variables

By default, the meta variables used to present schema data are prefixed with a `$` symbol. The individual meta variables are:

- `$count`
- `$prob`
- `$type`
- `$stats`
- `$hist`
- `$array`

The reason for the `$`-prefix is to distinguish any meta fields from actual data fields. MongoDB's drivers prevent $-prefixed keys to be written to the server, therefore we can be sure not to overwrite any user data. 

However, if the schema is going to be stored in MongoDB itself, then the names have to change, for above reasons. This can be achieved with the `metavars` option. Here is an example: 

```js

schema([
    {a: 1},
    {a: [-2, -3]}
], { 
    data: true, 
    metavars: { 
        prefix: "#", 
        count: "num", 
        stats: "statistics"
    } 
})

// output
{
    "#num": 2,
    "a": {
        "#num": 3,
        "#type": "number",
        "#statistics": {
            "min": -3,
            "max": 1
        },
        "#array": true,
        "#prob": 1.5
    }
}
```


#### Merge Existing Schema

Sometimes you want to merge an existing schema with some new data. In that case, you can pass in the existing schema, and it will be amended with the new values. Use the `merge` option to pass in an existing schema, like so:

```js
schema( documents, {merge: myExistingSchema} )
```

This works fine when the `data` option is not set, but if you infer data as well, this is not going to work, because the cleanup step throws away histograms of non-categorical data. If you want to merge a schema and also infer data, the best way is to use the `raw` mode. This mode returns the schema before the cleanup step. You can pass the raw schema back into another call to merge.

To clean the raw data up and convert it to a "final" version, just call the `.cleanup()` function on the raw schema object. 

Example:

```js
var raw_schema = schema( documents, {raw: true, data: true});
raw_schema = schema( more_documents, {raw; true, merge: raw_schema});
raw_schema = schema( even_more_documents, {raw; true, merge: raw_schema});
var schema = raw_schema.cleanup();
```


