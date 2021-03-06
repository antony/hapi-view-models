## hapi View Models

[![JavaScript Style Guide](https://img.shields.io/badge/code_style-standard-brightgreen.svg)](https://standardjs.com)
[![Build Status](https://circleci.com/gh/vendigo-group/hapi-view-models.png)](https://circleci.com)
[![Vendigo Open Source](https://img.shields.io/badge/vendigo-oss-brightgreen.svg)](http://github.com/vendigo-group)

A plugin to provide a concept of 'view-models' to [hapi](https://hapijs.com).

### Purpose

When rendering payloads from an API, we often want to provide different subsets (or supersets) of data to users with different *roles*, or *scopes*. By leveraging `lodash.omit` and `server.decorate`, we can contain the complexity of doing this as a fairly neat abstraction, reducing the overall complexity of payload modification and filtering, and keeping our controllers clean.

With `hapi-view-models` it is possible to render a variety of different versions of a single entity (payload) from a single endpoint.

### Installing

Installing is done in the usual way

```bash
npm install hapi-view-models
```

### Usage

The `vm` reply helper allows you to render views of your data:

```javascript
const KeyPairViewModel = require('...')
server.route({
  ...
  handler (request, reply) {
    reply.vm(KeyPairViewModel, response)
  }
})
```

Where vm takes the KeyPairViewModel and filters the data inside response according to the user's scope. Response can be a single entity or an array of entities.

If you're returning a response envelope you can provide a path to the entity as a third argument to the vm reply helper.

```javascript
const KeyPairViewModel = require('...')
server.route({
  ...
  handler (request, reply) {
    reply.vm(KeyPairViewModel, {
      nested: {
        data: [...]
      }
    }, 'nested.data')
  }
})
```

### Real World Example

Suppose we owned a cryptocurrency exchange. That's very 'of the moment', isn't it?

Lets say our wallet owner has a role of 'owner', and a visitor doesn't have this role.

We'd want to build a view model to render a wallet's data in a secure way. We declare a set of properties that are included in the payload for each role. Including a property in a role automatically **excludes that data from all other roles**.

Roles are defined by hapi in `request.auth.credentials.scope` and are controlled by your auth mechanism.

Our view model extends ViewModel and looks like this:

```javascript
const { ViewModel } = require('hapi-view-models')

class KeyPairViewModel extends ViewModel {
    get includes () {
      return {
        owner: ['private']
      }
    }
}
```

And our handler would look something like this:

```javascript
const { plugin } = require('hapi-view-models')
const KeyPairViewModel = require('path/to/key-pair-view-model')

// Register the plugin with hapi
server.register(plugin, err => {
  assert.ifError(err)
})

function getWalletKeys (address) {
  // Fetched from a database, more than likley.
  const keyPair = {
    private: '0x000',
    public: '0xaaa'
  }
}

server.route({
  method: 'GET',
  path: '/wallet/{address}',
  handler: function (request, reply) {
    const keyPair = getWalletKeys(request.params.address)
    reply.vm(KeyPairViewModel, keyPair)
  }
})
```

The rendered data would look something like this:

```javascript
// If the owner requests the wallet
{
  private: '0x000',
  public: '0xaaa'
}

// But if a visitor requests the wallet
{
  public: '0xaaa'
}
```

### Understanding `get includes()`

The getter method `get includes()` or just `.includes` as a property on your view model is a mapping of `scope` to an array of properties that are (deep) filtered from the resultant payload. This means we also support nesting!

This means you can do:

```javascript
const data = {
  a: {
    foo: 'bar'
  },
  b: {
    e: 'stuff here',
    c: {
      d: 'some-data'
    }
  }
}

class SomeViewModel extends ViewModel {
    get includes () {
      return {
        role1: ['a'], // Role 1 can see property 'a'
        role2: ['a', 'b'] // Role 2 can see property 'a' and 'b'
        role3: ['b.c.d'] // Without this role, you can see b and b.e, but the contents of b.c will be '{}' as 'b.c.d' is hidden.
      }
    }
}
```

You'll notice something going on here with includes. Includes is an 'exclusive' mapping. This means that if you declare a property visible by a role, **it is hidden for all other roles**. This means that you can minimally hide role-sensitive data, without having to re-iterate yourself or risk the leak of private properties leaking through.

TL;DR: Once a property is declared as *visible* to a role, it is automatically *invisible* to all other roles.

Ideally your user should have all the roles it needs to see all the data it needs, but if you like, you can 're-declare' visibility as we have done in the ase of `role2` above. A user wanting to see `a` can have roles `role1`, `role2`, or both. `a` (and as a result `a.foo`) isn't visible to any other roles.

### Structure

The plugin exports two modules:

  * `plugin` which is the `hapi` plugin providing `reply.vm()`
  * `ViewModel` which is the base class your view models should extend.

The plugin uses a slightly stricter extension of [standard-style](https://standardjs.com/)

### Contributing

To contribute to the plugin, fork it to your own github account, create a branch, make your changes, and submit a PR.

Note that Vendigo Finance Ltd requires that you include tests to cover your new code, along with your PR in order to get it merged.

To run our test suite:

```bash
npm install
npm test
npm run lint
```
