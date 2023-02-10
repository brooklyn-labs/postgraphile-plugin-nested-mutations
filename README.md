# postgraphile-plugin-nested-mutations

[![GitHub Workflow Status (with branch)](https://img.shields.io/github/actions/workflow/status/brooklyn-labs/postgraphile-plugin-nested-mutations/ci.yml?branch=master&label=Build)](https://github.com/brooklyn-labs/postgraphile-plugin-nested-mutations/actions/workflows/ci.yml)
[![npm (scoped)](https://img.shields.io/npm/v/@brooklyn-labs/postgraphile-plugin-nested-mutations?label=NPM)](https://www.npmjs.com/package/@brooklyn-labs/postgraphile-plugin-nested-mutations)
[![License](https://img.shields.io/github/license/brooklyn-labs/postgraphile-plugin-nested-mutations?label=License)](https://github.com/brooklyn-labs/postgraphile-plugin-nested-mutations/blob/master/LICENSE)

This plugin implements nested mutations based on both forward and reverse foreign
key relationships in PostGraphile v4.  Nested mutations can be of infinite depth.

## Getting Started

### CLI

``` bash
postgraphile --append-plugins postgraphile-plugin-nested-mutations
```

See [here](https://www.graphile.org/postgraphile/extending/#loading-additional-plugins) for
more information about loading plugins with PostGraphile.

### Library

``` js
const express = require('express');
const { postgraphile } = require('postgraphile');
const PostGraphileNestedMutations = require('postgraphile-plugin-nested-mutations');

const app = express();

app.use(
  postgraphile(pgConfig, schema, {
    appendPlugins: [
      PostGraphileNestedMutations,
    ],
  })
);

app.listen(5000);
```

### Plugin Options

When using PostGraphile as a library, the following plugin options can be passed 
via `graphileBuildOptions`:

<details>

<summary>nestedMutationsSimpleFieldNames</summary>

Use simple field names for nested mutations.  Instead of names suffixed with
`tableBy<Key>` and `tableUsing<Key>`, tables with a single foreign key relationship 
between them will have their nested relation fields named `table`.  Defaults to
`false`.

```js
postgraphile(pgConfig, schema, {
  graphileBuildOptions: {
    nestedMutationsSimpleFieldNames: true,
  }
});
```
</details>

<details>

<summary>nestedMutationsDeleteOthers</summary>

Controls whether the `deleteOthers` field is available on nested mutations.  Defaults
to `true`.

```js
postgraphile(pgConfig, schema, {
  graphileBuildOptions: {
    nestedMutationsDeleteOthers: false,
  }
});
```
</details>

<details>

<summary>nestedMutationsOldUniqueFields</summary>

If enabled, plural names for one-to-one relations will be used.  For backwards
compatibility.  Defaults to `false`.

```js
postgraphile(pgConfig, schema, {
  graphileBuildOptions: {
    nestedMutationsOldUniqueFields: false,
  }
});
```

</details>

## Usage

This plugin creates an additional field on each GraphQL `Input` type for every forward
and reverse foreign key relationship on a table, with the same name as the foreign table.

Each nested mutation field will have the following fields. They will accept an array if
the relationship is a one-to-many relationship, or a single input if they are one-to-one.

### Connect to Existing Record
#### `connectByNodeId`
Connect using a `nodeId` from the nested table.

#### `connectBy<K>`
Connect using any readable primary key or unique constraint on the nested table.

### Creating New Records
#### `create`
Create a new record in the nested table.

### Delete existing Record
#### `deleteByNodeId`
Delete using a `nodeId` from the nested table.

#### `deleteBy<K>`
Delete using any readable primary key or unique constraint on the nested table.

### Updating Records
#### `updateByNodeId`
Update a record using a `nodeId` from the nested table.

#### `updatedBy<K>`
Update a record using any readable primary key or unique constraint on the nested table.

## Example

```sql
create table parent (
  id serial primary key,
  name text not null
);

create table child (
  id serial primary key,
  parent_id integer,
  name text not null,
  constraint child_parent_fkey foreign key (parent_id)
    references p.parent (id)
);
```

A nested mutation against this schema, using `Parent` as the base mutation
would look like this:

``` graphql
mutation {
  createParent(input: {
    parent: {
      name: "Parent 1"
      childrenUsingId: {
        connectById: [{
          id: 1
        }]
        create: [{
          name: "Child 1"
        }, {
          name: "Child 2"
        }]
      }
    }
  }) {
    parent {
      id
      name
      childrenByParentId {
        nodes {
          id
          name
        }
      }
    }
  }
}
```

Or using `Child` as the base mutation:

``` graphql
mutation {
  createChild(input: {
    child: {
      name: "Child 1"
      parentToParentId: {
        create: {
          name: "Parent of Child 1"
        }
      }
    },
  }) {
    child {
      id
      name
      parentByParentId {
        id
        name
      }
    }
  }
}
```

## Smart Comments

[Smart comments](https://www.graphile.org/postgraphile/smart-comments/) are supported for 
renaming the nested mutation fields.

```sql
comment on constraint child_parent_fkey on child is
  E'@fieldName parent\n@foreignFieldName children';
```
