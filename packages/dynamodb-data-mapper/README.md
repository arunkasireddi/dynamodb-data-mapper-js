# Amazon DynamoDB DataMapper

[![Apache 2 License](https://img.shields.io/github/license/awslabs/dynamodb-data-mapper-js.svg?style=flat)](http://aws.amazon.com/apache-2-0/)

This library provides a `DataMapper` class that allows easy interoperability
between your application's domain classes and their persisted form in Amazon
DynamoDB. Powered by the `@aws/dynamodb-data-marshaller` and
`@aws/dynamodb-expressions` packages, using `DataMapper` lets you define each
object's persisted representation once and then load, save, scan, and query your
tables using the vocabulary of your application domain rather than its
representation in DynamoDB.

## Getting started

To use the `DataMapper` with a given JavaScript class, you will need to add a
couple properties to the prototype of the class you would like to map to a
DynamoDB table. Specifically, you will need to provide a schema and the name of
the table:

```typescript
import {DynamoDbSchema, DynamoDbTable} from '@aws/dynamodb-data-mapper';

class MyDomainModel {
    // declare methods and properties as normal
}

Object.defineProperties(MyDomainModel.prototype, {
    [DynamoDbTable]: {
        value: 'MyTable'
    },
    [DynamoDbSchema]: {
        value: {
            id: {
                type: 'String',
                keyType: 'HASH'
            },
            foo: {type: 'String'},
            bar: {
                type: 'Set',
                memberType: 'String',
            },
            baz: {
                type: 'Tuple',
                members: [
                    {type: 'Boolean'},
                    {type: 'String'},
                ],
            },
        },
    },
});
```

The schema and table name may be declared as property accessors directly on the
class if the value should be determined dynamically:

```typescript
import {DynamoDbTable} from '@aws/dynamodb-data-mapper';

class MyOtherDomainClass {
    id: number;

    get [DynamoDbTable]() {
        return this.id % 2 === 0 ? 'evens' : 'odds';
    }
}
```

Next, create an instance of `DataMapper` and use the `MyDomainClass` constructor
defined above to save and load objects from DynamoDB:

```typescript
import {
    DataMapper,
    DynamoDbSchema,
    DynamoDbTable,
} from '@aws/dynamo-data-mapper';
import DynamoDB = require('aws-sdk/clients/dynamodb');

const client = new DynamoDB({region: 'us-west-2'});
const mapper = new DataMapper({client});

class MyDomainModel {
    id: string;
    foo?: string;
    bar?: Set<string>;
    baz?: [boolean, string];
}

Object.defineProperties(MyDomainModel.prototype, {
    [DynamoDbTable]: {
        value: 'MyTable'
    },
    [DynamoDbSchema]: {
        value: {
            id: {
                type: 'String',
                keyType: 'HASH'
            },
            foo: {type: 'String'},
            bar: {
                type: 'Set',
                memberType: 'String',
            },
            baz: {
                type: 'Tuple',
                members: [
                    {type: 'Boolean'},
                    {type: 'String'},
                ],
            },
        },
    },
});

// delete an object
const toDelete = new MyDomainModel();
toDelete.id = 'DELETE_ME';
mapper.delete(toDelete);

// if that's too verbose, you can write the above as a single expression with
// Object.assign:
mapper.delete(Object.assign(new MyDomainModel(), {id: 'DELETE_ME'}));

// fetch an object
const toGet = new MyDomainModel();
toGet.id = 'ID_TO_FETCH';
const fetched = await mapper.get(toGet);

// this should return a rejected promise, as it's fetching an object that does
// not exist
mapper.get(toDelete)
    .catch(err => console.log('I expected this to happen'));

// put something new into the database
const toPut = new MyDomainModel();
toPut.id = 'NEW_RECORD';
toPut.foo = 'bar';
toPut.bar = new Set<string>(['fizz', 'buzz', 'pop']);
toPut.baz = [true, 'quux'];

mapper.put(toPut).then((persisted: MyDomainModel) => {
    // now change the record a bit
    const toUpdate = new MyDomainModel();
    toUpdate.id = persisted.id;
    toUpdate.baz = [false, 'beep'];
    return mapper.update(toUpdate, {onMissing: 'skip'});
});
```

## Supported operations

### `delete`

Removes an item from a DynamoDB table. Supports the following named parameters:

* `item` - (**Required**) The item to be deleted. Must be an instance of a class
    with a table name accessible via a property identified with the
    `DynamoDbTable` symbol and a schema accessible via a property identified
    with the `DynamoDbSchema` symbol.

* `condition` - A condition expression whose assertion must be satisfied in
    order for the delete operation to be executed. Please refer to the
    documentation for the `@aws/dynamodb-expressions` package for guidance on
    creating condition expression objects.

* `returnValues` - Specify `'ALL_OLD'` to have the deleted item returned to you
    when the delete operation completes.

* `skipVersionCheck` - Whether to forgo creating a condition expression based on
    a defined `versionAttribute` in the schema.

### `get`

Fetches an item from a DynamoDB table. If no item with the specified key was
found, the returned promise will be rejected with an error. Supports the
following named parameters:

* `item` - (**Required**) The item to be deleted. Must be an instance of a class
    with a table name accessible via a property identified with the
    `DynamoDbTable` symbol and a schema accessible via a property identified
    with the `DynamoDbSchema` symbol.

    The supplied item will **NOT** be updated in place. Rather, a new item of
    the same class with data from the DynamoDB table will be returned.

* `readConsistency` - Specify `'strong'` to perform a strongly consistent read.
    Specify `'eventual'` (the default) to perform an eventually consistent read.

* `projection` - A projection expression directing DynamoDB to return a subset
    of the fetched item's attributes. Please refer to the documentation for the
    `@aws/dynamodb-expressions` package for guidance on creating projection
    expression objects.

### `put`

Inserts an item into a DynamoDB table. Supports the following named parameters:

* `item` - (**Required**) The item to be inserted. Must be an instance of a
    class with a table name accessible via a property identified with the
    `DynamoDbTable` symbol and a schema accessible via a property identified
    with the `DynamoDbSchema` symbol.

* `condition` - A condition expression whose assertion must be satisfied in
    order for the put operation to be executed. Please refer to the
    documentation for the `@aws/dynamodb-expressions` package for guidance on
    creating condition expression objects.

* `returnValues` - Specify `'ALL_OLD'` to have the overwritten item (if one
    existed) returned to you when the put operation completes.

* `skipVersionCheck` - Whether to forgo creating a condition expression based on
    a defined `versionAttribute` in the schema.

### `query`

Retrieves multiple values from a table based on the primary key attributes.
Queries must target a single partition key value but may read multiple items
with different range keys.

This method is implemented as an async iterator and the results can be consumed
with a `for-await-of` loop. If you are using TypeScript, you will need to
include `esnext.asynciterable` in your `lib` declaration (as well as enabling
`downlevelIteration` if targeting ES5 or lower). Please refer to [the TypeScript
release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-3.html#async-iteration)
for more information.

* `keyCondition` - (**Required**) The condition that specifies the key value(s)
    for items to be retrieved by the query operation. You may provide a hash
    matching key properties to the values they must equal, a hash matching keys
    to `ConditionExpressionPredicate`s, or a fully composed
    `ConditionExpression`. If a hash is provided, it may contain a mixture of
    condition expression predicates and exact value matches:

    ```typescript
    import {between} from '@aws/dynamodb-expressions';

    const keyCondition = {
        partitionKey: 'foo',
        rangeKey: between(10, 99),
    };
    ```

    The key condition must target a single value for the partition key.

    Please refer to the documentation for the `@aws/dynamodb-expressions`
    package for guidance on creating condition expression objects.

* `valueConstructor` - (**Required**) The constructor to use for any results
    returned by this operation. Must have a prototype with a table name
    accessible via a property identified with the `DynamoDbTable` symbol and a
    schema accessible via a property identified with the `DynamoDbSchema`
    symbol.

* `filter` - A condition expression that DynamoDB applies after the Query
    operation, but before the data is returned to you. Items that do not satisfy
    the `filter` criteria are not returned.

    You cannot define a filter expression based on a partition key or a sort
    key.

    Please refer to the documentation for the `@aws/dynamodb-expressions`
    package for guidance on creating condition expression objects.

* `indexName` - The name of the index against which to execute this query. If
    not specified, the query will be executed against the base table.

* `limit` - The maximum number of items to return.

* `projection` - A projection expression directing DynamoDB to return a subset
    of any fetched item's attributes. Please refer to the documentation for the
    `@aws/dynamodb-expressions` package for guidance on creating projection
    expression objects.

* `readConsistency` - Specify `'strong'` to perform a strongly consistent read.
    Specify `'eventual'` (the default) to perform an eventually consistent read.

* `scanIndexForward` - Specifies the order for index traversal: If true, the
    traversal is performed in ascending order; if false, the traversal is
    performed in descending order.

* `startKey` - The primary key of the first item that this operation will
    evaluate.

### `scan`

Retrieves all values in a table.

This method is implemented as an async iterator and the results can be consumed
with a `for-await-of` loop. If you are using TypeScript, you will need to
include `esnext.asynciterable` in your `lib` declaration (as well as enabling
`downlevelIteration` if targeting ES5 or lower). Please refer to [the TypeScript
release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-3.html#async-iteration)
for more information.

* `valueConstructor` - (**Required**) The constructor to use for any results
    returned by this operation. Must have a prototype with a table name
    accessible via a property identified with the `DynamoDbTable` symbol and a
    schema accessible via a property identified with the `DynamoDbSchema`
    symbol.

* `filter` - A condition expression that DynamoDB applies after the scan
    operation, but before the data is returned to you. Items that do not satisfy
    the `filter` criteria are not returned.

    You cannot define a filter expression based on a partition key or a sort
    key.

    Please refer to the documentation for the `@aws/dynamodb-expressions`
    package for guidance on creating condition expression objects.

* `indexName` - The name of the index against which to execute this query. If
    not specified, the query will be executed against the base table.

* `limit` - The maximum number of items to return.

* `projection` - A projection expression directing DynamoDB to return a subset
    of any fetched item's attributes. Please refer to the documentation for the
    `@aws/dynamodb-expressions` package for guidance on creating projection
    expression objects.

* `readConsistency` - Specify `'strong'` to perform a strongly consistent read.
    Specify `'eventual'` (the default) to perform an eventually consistent read.

* `segment` - The identifier for this segment (if this scan is being performed
    as part of a parallel scan operation).

* `startKey` - The primary key of the first item that this operation will
    evaluate.

* `totalSegments` - The number of segments into which this scan has been divided
    (if this scan is being performed as part of a parallel scan operation).

### `parallelScan`

Retrieves all values in a table by dividing the table into segments, all of
which are scanned in parallel.

This method is implemented as an async iterator and the results can be consumed
with a `for-await-of` loop. If you are using TypeScript, you will need to
include `esnext.asynciterable` in your `lib` declaration (as well as enabling
`downlevelIteration` if targeting ES5 or lower). Please refer to [the TypeScript
release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-3.html#async-iteration)
for more information.

* `valueConstructor` - (**Required**) The constructor to use for any results
    returned by this operation. Must have a prototype with a table name
    accessible via a property identified with the `DynamoDbTable` symbol and a
    schema accessible via a property identified with the `DynamoDbSchema`
    symbol.

* `segments` - (**Required**) The total number of parallel workers to use to
    scan the table.

* `filter` - A condition expression that DynamoDB applies after the scan
    operation, but before the data is returned to you. Items that do not satisfy
    the `filter` criteria are not returned.

    You cannot define a filter expression based on a partition key or a sort
    key.

    Please refer to the documentation for the `@aws/dynamodb-expressions`
    package for guidance on creating condition expression objects.

* `indexName` - The name of the index against which to execute this query. If
    not specified, the query will be executed against the base table.

* `limit` - The maximum number of items to return.

* `projection` - A projection expression directing DynamoDB to return a subset
    of any fetched item's attributes. Please refer to the documentation for the
    `@aws/dynamodb-expressions` package for guidance on creating projection
    expression objects.

* `readConsistency` - Specify `'strong'` to perform a strongly consistent read.
    Specify `'eventual'` (the default) to perform an eventually consistent read.

* `startKey` - The primary key of the first item that this operation will
    evaluate.

### `update`

Updates an item in a DynamoDB table. Will leave attributes not defined in the
schema in place.

* `item` - (**Required**) The item with its desired property state. Must be an
    instance of a class with a table name accessible via a property identified
    with the `DynamoDbTable` symbol and a schema accessible via a property
    identified with the `DynamoDbSchema` symbol.

* `condition` - A condition expression whose assertion must be satisfied in
    order for the update operation to be executed. Please refer to the
    documentation for the `@aws/dynamodb-expressions` package for guidance on
    creating condition expression objects.

* `onMissing` - Specify `'remove'` (the default) to treat the absence of a value
    in the supplied `item` as a directive to remove the property from the record
    in DynamoDB. Specify `'skip'` to only update the properties that are defined
    in the supplied `item`.

* `skipVersionCheck` - Whether to forgo creating a condition expression based on
    a defined `versionAttribute` in the schema.
