# Ember Style Guide

## Table of contents:
* [General](#general)
* [Organizing](#organizing)
* [Controllers](#controllers)
* [Components](#components)
* [Ember Data](#ember-data)
* [Routing](#routing)
* [Templates](#templates)
* [Tests](#tests)

## General

### Create local version of Ember.* and DS.*
Ember can use new functionality of ES6 - `modules`. In near future, Ember will be build using this convention and eventually we will import `computed` instead of `Ember.computed`. To make code more clear and ready for future, we can create local versions of these modules.
```javascript
import Ember from 'ember';
import DS from 'ember-data';

// GOOD
const { Model, attr } = DS;
const { computed } = Ember;
const { alias } = computed;

export default Model.extend({
  name: attr('string'),
  degree: attr('string'),
  title: alias('degree'),

  fullName: computed('name', 'degree' function() {
    return `${this.get('degree')} ${this.get('name')}`;
  }),
});

// BAD
export default DS.Model.extend({
  name: DS.attr('string'),
  degree: DS.attr('string'),
  title: Ember.computed.alias('degree'),

  fullName: Ember.computed('name', 'degree', {
    return `${this.get('degree')} ${this.get('name')}`;
  }),
});
```

### Don’t use jQuery without Ember Run Loop
Using plain jQuery provides invoke actions out of the Ember Run Loop. To have control on all operations in Ember it's good practice to trigger actions in run loop.
```javascript
/// GOOD
Ember.$('#something-rendered-by-jquery-plugin').on('click', () => {
  Ember.run.bind(this, this._handlerActionFromController);
});

// BAD
Ember.$('#something-rendered-by-jquery-plugin').on('click', () => {
  this._handlerActionFromController();
});
```


### Don't use observers
Usage of observers is very easy **BUT** it leads to hard to reason about consequences. If observers are not necessary then better to avoid them.
```hbs
{{input value=text key-up="change"}}
```

```javascript
// GOOD
export default Controller.extend({
  actions: {
    change() {
      console.log(`change detected: ${this.get('text')}`);
    },
  },
});

// BAD
export default Model.extend({
  change: Ember.observer('text', function() {
    console.log(`change detected: ${this.get('text')}`);
  },
});
```

### Don't introduce side-effects in computed properties
When using computed properties do not introduce side effects. It will make reasoning about the origin of the change much harder.

```js
import Ember from 'ember';

const {
  Component,
  computed: { filterBy, alias },
} = Ember;

export default Component.extend({
  users: [
    { name: 'Foo', age: 15 },
    { name: 'Bar', age: 16 },
    { name: 'Baz', age: 15 }
  ],

  // GOOD:
  fifteen: filterBy('users', 'age', 15),
  fifteenAmount: alias('fifteen.length'),

  // BAD:
  fifteenAmount: 0,
  fifteen: computed('users', function() {
    const fifteen = this.get('users').filterBy('items', 'age', 15);
    this.set('fifteenAmount', fifteen.length); // SIDE EFFECT!
    return fifteen;
  })
});
```

### Use named functions defined on objects to handle promises
When you use promises and its handlers, use named functions defined on parent object. Thus, you will be able to test them in isolation using unit tests without any additional mocking.

```js
export default Component.extend({
  actions: {
    // BAD
    updateUser(user) {
      user.save().then(() => {
        return user.reload();
      }).then(() => {
        this.notifyAboutSuccess();
      }).catch(() => {
        this.notifyAboutFailure();
      });
    },
    // GOOD
    updateUser(user) {
      user.save()
        .then(this._reloadUser.bind(this))
        .then(this._notifyAboutSuccess.bind(this))
        .catch(this._notifyAboutFailure.bind(this));
    },
  },
  _reloadUser(user) {
    return user.reload();
  },
  _notifyAboutSuccess() {
    // ...
  },
  _notifyAboutFailure() {
    // ...
  },
});
```

And then you can make simple unit tests for handlers:
```
test('it reloads user in promise handler', function(assert) {
  const component = this.subject();
  // assuming that you have `user` defined with kind of sinon spy on its reload method
  component._reloadUser(user);
  assert.ok(userReloadSpy.calledOnce, 'user#reload should be called once');
});
```

## Organizing

### Organize your components
To maintain good readable of code, you should write code grouped and ordered in this way:
1. Default values
2. Single line computed properties
3. Multiline computed properties
4. Actions

```javascript
const { Component, computed } = Ember;
const { alias } = computed;

export default Component.extend({
  // 1. Defaults
  role: 'sloth',

  // 2. Single line Computed Property
  vehicle: alias('car'),

  // 3. Multiline Computed Property
  levelOfHappiness: computed('attitude', 'health', {
    const result = this.get('attitude') * this.get('health') * Math.random();
    return result;
  }),

  // 4. All actions
  actions: {
    sneakyAction() {
      return this._secretMethod();
    }
  }
});
```

### Organize your models
Build model groups for each type of element. You should create 3 main subgroups in this order:
1. Attributes
2. Relations
3. Computed Properties

```javascript
// GOOD
export default Model.extend({
  // 1. Attributes
  shape: attr('string'),

  // 2. Relations
  behaviors: hasMany('behaviour'),

  // 3. Computed Properties
  mood: computed('health', 'hunger', {
    const result = this.get('health') * this.get('hunger');
    return result;
  })
});

// BAD
export default Model.extend({
  mood: computed('health', 'hunger', {
    const result = this.get('health') * this.get('hunger');
    return result;
  }),

  hat: attr('string'),

  behaviors: hasMany('behaviour'),

  shape: attr('string')
});
```

### Use PODs structure
Standard file structure in Ember App is divided by type of file function. Pods organize files by features, it's much better solution because in bigger project finding particular file is significant faster. But whether everything should be kept in pods?

- what includes in pods:
  - Routes
  - Components
  - Controllers
  - Templates

- what not includes in pods:
  - Models

```
// GOOD
app
  models/
    plants.js
    chemicals.js
  pods/
    application/
      controller.js
      route.js
      template.hbs
    login/
      controller.js
      route.js
      templates.hbs
    plants/
      controller.js
      route.js
      template.hbs
    components/
      displayOfDanger
        component.js
        template.hbs
```

## Controllers

### Alias your model
It makes code more readable if model has the same name as a subject. It’s more maintainable, and will conform to future  routable components. We can do this in two ways:
- set alias to model (in case when there is a `Nail Controller`):
```javascript
const { alias } = Ember.computed;
export default Ember.Controller.extend({
  nail: alias('model'),
});
```
- set it in `setupController` method:
```javascript
export default Ember.Route.extend({
  setupController(controller, model) => {
    controller.set('nail', model);
  },
});
```

### Query params should always be on top
If you are using query params in your controller, those should always be placed on top. It will make spotting them much easier.

```js
import Ember from 'ember';

const { Controller} = Ember;

// BAD
export default Controller.extend({
  statusOptions: Ember.String.w('Accepted Pending Rejected')
  status: [],
  queryParams: ['status'],
});

// GOOD
export default Controller.extend({
  queryParams: ['status'],
  status: [],

  statusOptions: Ember.String.w('Accepted Pending Rejected')
});
```

## Ember Data

### Be explicit with Ember data attribute types
Ember Data could handle lack of specified types in model description. Nonetheless this could lead to ambiguity. Therefore always supply proper attribute type to ensure the right data transform is used.
```javascript
const { Model, attr } = DS;

// GOOD
export default Model.extend({
  name: attr('string'),
  points: attr('number'),
  dob: attr('date'),
});

// BAD
export default Model.extend({
  name: attr(),
  points: attr(),
  dob: attr(),
});
```

In case when you need a custom behavior it's good to write own [Transform](http://emberjs.com/api/data/classes/DS.Transform.html)


## Components

### Data Down Action Up
You should't change passed data in components instead trigger actions that should change this data.

```hbs
{{! GOOD }}
{{nice-component dataArray=data removeElement=(action "removeElement")}}
```
```javascript
export default Component.extend({
  actions: {
    removeElement(element) {
      this.attr.removeElement(element);
    },
  },
});

export default Controller.extend({
  actions: {
    removeElement(element) {
      this.get('data').removeObject(element);
    },
  },
});
```

```hbs
{{! BAD }}
{{uggly-component dataArray=data}}
```
```javascript
export default Component.extend({
  actions: {
    removeElement(element) {
      this.get('dataArray').removeObject(element);
    },
  },
});
```

### Service-backed Components
Sometimes you have some data that are not crucial for given page and can be loaded after the page has been rendered. This way you don't block the page from rendering until those data have been fetched from the API. Post comments are a good example. You definitely need post content before the page is rendered - but not necessarily comments. Those do not feel like an appropriate concern for the router and so we can fetch those in a service-backed components, e.g:

```js
import Ember from 'ember';

const {
  Component,
  inject: { service }
} = Ember;

export default Component.extend({
  store: service(),
  loadComments() {
    this.get('store').findAll('comment').then((comments) => {
      // do something with the comments
    });
  }
});
```

Summarizing: if you want to change the URL or given data is required for the loaded page to make sense - load data in the router. Otherwise you might want to use a service-backed component.

### Closure Actions
Always use closure actions (according to DDAU convention). Exception: only when you need bubbling.

```javascript
export default Controller.extend({
  actions: {
    detonate() {
      alert('Kabooom');
    }
  }
});
```

```hbs
{{! GOOD }}
{{pretty-component boom=(action 'detonate')}}
```

```javascript
export default Component.extend({
  actions: {
    pushLever() {
      this.attr.boom();
    }
  }
})
```

```hbs
{{! BAD }}
{{awful-component detonate='detonate'}}
```
```javascript
export default Component.extend({
  actions: {
    pushLever() {
      this.sendAction('detonate');
    }
  }
})
```
## Routing

### Route naming
Dynamic segments in routes should use _snake case_. Reason:
- Ember could resolve promises without extra serialization work

```javascript
// GOOD
this.route('tree', { path: ':tree_id'});

// BAD
this.route('tree', { path: ':treeId'});
```

## Templates

## Use components in `{{#each}}` blocks
When content of each block is larger than one line, use component to wrap this code. Ember convention is build app through divided to smaller modules (Components). This is more flexible and readable. Use simple rule of thumb - if you need more than one line in `#each` block, then use component.

```hbs
{{! GOOD }}
{{#each paintings as |paiting|}}
  {{paiting-details paiting=paiting}}
{{/each}}

{{! BAD }}
{{#each paintings as |paiting|}}
  <title>{{paiting.title}}</title>
  <author>{{paiting.author}}</author>
  <img src={{paiting.image}}>
  <div>{{paiting.cost}}<div>
{{/each}}
```

## Tests

### Use page objects in acceptance testing
In acceptance tests there is a tendency to repeat the same code meny times (mainly selectors). It's also hard to reason about exact behavior based only on selector naes. To avoid this we can extract this to other abstract layer and then import them to tests.

```javascript
// GOOD
export default Ember.Object({
  expectGroceryHeader(msg) {
    andThen(() => {
      this.get('assert').equal(find('#grocery-header').text(), msg);
    });
    return this;
  }
})

import PageObject from 'my-project/tests/page-objects/base';
...
test('check changed message', function(assert) {
  PageObject.create({ assert })
            .expectGroceryHeader('cabbage');
});
```
