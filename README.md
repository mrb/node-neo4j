# Node-Neo4j

[![npm version](https://badge.fury.io/js/neo4j.svg)](http://badge.fury.io/js/neo4j) [![Build Status](https://travis-ci.org/thingdom/node-neo4j.svg?branch=master)](https://travis-ci.org/thingdom/node-neo4j)

[Node.js](http://nodejs.org/) driver for [Neo4j](http://neo4j.com/), a graph database.

This driver aims to be the most **robust**, **comprehensive**, and **battle-tested** driver available. It's run in production by [FiftyThree](https://www.fiftythree.com/) to power [Paper](https://www.fiftythree.com/paper) and [Mix](https://mix.fiftythree.com/).


## Features

- **Cypher queries**, parameters, and **transactions**
- Arbitrary **HTTP requests**, for custom **Neo4j plugins**
- **Custom headers**, for **high availability**, application tracing, query logging, and more
- **Precise errors**, for robust error handling from the start
- Configurable **connection pooling**, for performance tuning & monitoring
- Thorough test coverage with **>100 tests**
- **Continuously integrated** against **multiple versions** of Node.js and Neo4j


## Installation

```sh
npm install neo4j --save
```


## Example

```js
var neo4j = require('neo4j');
var db = new neo4j.GraphDatabase('http://username:password@localhost:7474');

db.cypher({
    query: 'MATCH (user:User {email: {email}}) RETURN user',
    params: {
        email: 'alice@example.com',
    },
}, callback);

function callback(err, results) {
    if (err) throw err;
    var result = results[0];
    if (!result) {
        console.log('No user found.');
    } else {
        var user = result['user'];
        console.log(JSON.stringify(user, null, 4));
    }
};
```

Yields e.g.:

```json
{
    "_id": 12345678,
    "labels": [
        "User",
        "Admin"
    ],
    "properties": {
        "name": "Alice Smith",
        "email": "alice@example.com",
        "emailVerified": true,
        "passwordHash": "..."
    }
}
```

See [node-neo4j-template](https://github.com/aseemk/node-neo4j-template) for a more thorough example.

TODO: Also link to movies example.


## Basics

Connect to a running Neo4j instance by instantiating the **`GraphDatabase`** class:

```js
var neo4j = require('neo4j');

// Shorthand:
var db = new neo4j.GraphDatabase('http://username:password@localhost:7474');

// Full options:
var db = new neo4j.GraphDatabase({...});
```

Options:

- **`url` (required)**: the base URL to the Neo4j instance, e.g. `'http://localhost:7474'`. This can include auth credentials (e.g. `'http://username:password@localhost:7474'`), but doesn't have to.

- **`auth`**: optional auth credentials; either a `'username:password'` string, or a `{username, password}` object. If present, this takes precedence over any credentials in the `url`.

- **`headers`**: optional custom HTTP headers to send with every request. These can be overridden per request. Node-Neo4j defaults to sending a `User-Agent` identifying itself, but this can be overridden too.

- **`proxy`**: optional URL to a proxy. If present, all requests will be routed through the proxy.

- **`agent`**: optional [`http.Agent`](http://nodejs.org/api/http.html#http_http_agent) instance, for custom socket pooling.

Once you have a `GraphDatabase` instance, you can make queries and more.

Most operations are **asynchronous**, which means they take a **callback**. Node-Neo4j callbacks are of the standard `(error[, results])` form.

Async control flow can get tricky quickly, so it's *highly* recommended to use a flow control library or tool, like [async](https://github.com/caolan/async) or [Streamline](https://github.com/Sage/streamlinejs).


## Cypher

To make a Cypher query, simply pass the string query, any query parameters, and a callback to receive the error or results.

```js
db.cypher({
    query: 'MATCH (user:User {email: {email}}) RETURN user',
    params: {
        email: 'alice@example.com',
    },
}, callback);
```

It's extremely important to pass `params` separately. If you concatenate them into the `query`, you'll be vulnerable to injection attacks, and Neo4j performance will suffer as well.

Cypher queries *always* return a list of results (like SQL rows), with each result having common properties (like SQL columns). Thus, query **results** passed to the callback are *always* an **array** (even if it's empty), and each **result** in the array is *always* an **object** (even if it's empty).

```js
function callback(err, results) {
    if (err) throw err;
    var result = results[0];
    if (!result) {
        console.log('No user found.');
    } else {
        var user = result['user'];
        console.log(JSON.stringify(user, null, 4));
    }
};
```

If the query results include nodes or relationships, **`Node`** and **`Relationship`** instances are returned for them. These instances encapsulate `{_id, labels, properties}` for nodes, and `{_id, type, properties, _fromId, _toId}` for relationships, but they can be used just like normal objects.

```json
{
    "_id": 12345678,
    "labels": [
        "User",
        "Admin"
    ],
    "properties": {
        "name": "Alice Smith",
        "email": "alice@example.com",
        "emailVerified": true,
        "passwordHash": "..."
    }
}
```

(The `_id` properties refer to Neo4j's internal IDs. These can be convenient for debugging, but their use otherwise — especially externally — is discouraged.)

If you don't need to know Neo4j IDs, node labels, or relationship types, you can pass `lean: true` to get back *just* properties, for a potential performance gain.

```js
db.cypher({
    query: 'MATCH (user:User {email: {email}}) RETURN user',
    params: {
        email: 'alice@example.com',
    },
    lean: true,
}, callback);
```

```json
{
    "name": "Alice Smith",
    "email": "alice@example.com",
    "emailVerified": true,
    "passwordHash": "..."
}
```

Other options:

- **`headers`**: optional custom HTTP headers to send with this query. These will add onto the default `GraphDatabase` `headers`, but also override any that overlap.


## Batching

Although this need is rare, you can make multiple Cypher queries in a single network request, by passing a `queries` *array* rather than a single `query` string.

Query `params` (and optionally `lean`) are then specified *per query*, so the elements in the array are `{query, params[, lean]}` objects. (Other options like `headers` remain "global" for the entire request.)

```js
db.cypher({
    queries: [{
        query: 'MATCH (user:User {email: {email}}) RETURN user',
        params: {
            email: 'alice@example.com',
        },
    }, {
        query: 'MATCH (task:WorkerTask) RETURN task',
        lean: true,
    }, {
        query: 'MATCH (task:WorkerTask) DELETE task',
    }],
    headers: {
        'X-Request-ID': '1234567890',
    },
}, callback);
```

The callback then receives an *array* of query results, one per query.

```js
function callback(err, batchResults) {
    if (err) throw err;

    var userResults = batchResults[0];
    var taskResults = batchResults[1];
    var deleteResults = batchResults[2];

    // User results:
    var userResult = userResults[0];
    if (!userResult) {
        console.log('No user found.');
    } else {
        var user = userResult['user'];
        console.log('User %s (%s) found.', user._id, user.properties.name);
    }

    // Worker task results:
    if (!taskResults.length) {
        console.log('No worker tasks to process.');
    } else {
        taskResults.forEach(function (taskResult) {
            var task = taskResult['task'];
            console.log('Processing worker task %s...', task.operation);
        });
    }

    // Delete results (shouldn’t have returned any):
    assert.equal(deleteResults.length, 0);
};
```

Importantly, batch queries execute (a) sequentially and (b) transactionally: either they all succeed, or they all fail. If you don't need them to be transactional, it can often be better to parallelize separate `db.cypher` calls instead.



<!--

## Getting Help

If you're having any issues you can first refer to the [API documentation][api-docs].

If you encounter any bugs or other issues, please file them in the
[issue tracker][issue-tracker].

We also now have a [Google Group][google-group]!
Post questions and participate in general discussions there.

You can also [ask a question on StackOverflow][stackoverflow-ask]

-->


## Changes

See the [Changelog][changelog] for the full history of changes and releases.


## License

This library is licensed under the [Apache License, Version 2.0][license].



[node.js]: http://nodejs.org/
[neo4j-rest-api]: http://docs.neo4j.org/chunked/stable/rest-api.html

[api-docs]: http://coffeedoc.info/github/thingdom/node-neo4j/master/
[aseemk]: https://github.com/aseemk
[node-neo4j-template]: https://github.com/aseemk/node-neo4j-template
[semver]: http://semver.org/

[coffeescript]: http://coffeescript.org/
[streamline.js]: https://github.com/Sage/streamlinejs

[changelog]: CHANGELOG.md
[issue-tracker]: https://github.com/thingdom/node-neo4j/issues
[license]: http://www.apache.org/licenses/LICENSE-2.0.html
[google-group]: https://groups.google.com/group/node-neo4j

[stackoverflow-ask]: http://stackoverflow.com/questions/ask?tags=node.js,neo4j,thingdom
