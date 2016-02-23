# Ember Style Guide

## Table of contents:
* [General](#general)
* [Organize](#organize)
* [Controllers](#controllers)
* [Components](#components)
* [Ember Data](#ember-data)
* [Tests](#tests)

## General

### Create local version of Ember.* and DS.*
Ember could perfectly use a new functionality of ES6 namely `modules`. To make code more clear we are able to import e.g. `Ember.computed` directly as `computed`. This operation reduces unneeded namespaces.
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
    })
});

// BAD
export default DS.Model.extend({
	name: DS.attr('string'),
    degree: DS.attr('string'),  
    title: Ember.computed.alias('degree'),

    fullName: Ember.computed('name', 'degree', {
    	get() {
        	// code
        }
        set() {
        	// code
        }
    }),

    wholeName: function() {
    	return this.get('degree') + ' ' + this.get('name');
    }.property('name', 'degree')
});
```

### Don’t use jQuery without Ember Run Loop
Using plain jQuery provides invoke actions out of the App. Ember App is based on run loop which is a main handler of all actions. To have control on all operations in ember it's good practice to trigger action in run loop.
```javascript
/// GOOD
Ember.run.next(this, () => {
	Ember.$('#message').text('Bazinga');
});

// BAD
Ember.$('#message').text('Bazinga');
/// Ember.$ is only an alias to jQuery
```


### Don't use observers
Usage observers is very easy **BUT** it leads to unpredictable actions in app. Observers are hard to analyze because they do jobs on background. If observers are not necessary then better to omit them.
```hbs
{{input value=text key-up="change"}}
```

```javascript
// GOOD
export default Controller.extend({
	actions: {
    	change() {
        	console.log(`change detected: ${this.get('text')}`);
        }
    }
});

// BAD
export default Model.extend({
	change: Ember.observer('text', function() {
    	console.log('change detected: ' + this.get('text'));
    }
});
```

## Controllers

### Alias your model
It makes code more readable if model has the same name as a subject. It’s more maintainable, and will conform to future  routable components. We can do this in two ways:
- set alias to model (in case when there is a `Nail Controller`):
```javascript
const { alias } = Ember.computed;
export default Ember.Controller.extend({
	nail: alias('model')
});
```
- set it in `setupController` method:
```javascript
export default Ember.Route.extend({
	setupController(controller, model) => {
    	controller.set('nail', model);
    }
});
```

## Ember Data

### Be explicit with Ember data attribute types
Ember Data could handle lack of specified types in model description. Nonetheless this could lead to ambiguity. Therefore always supply proper an attribute type to ensure the right data transform is used.
```javascript
const { Model, attr } = DS;

// GOOD
export default Model.extend({
	name: attr('string'),
    points: attr('number'),
    dob: attr('date')
});

// BAD
export default Model.extend({
	name: attr(),
    points: attr(),
    dob: attr()
});
```

In case when you need a custom behavior it's good to write own [Transform](http://emberjs.com/api/data/classes/DS.Transform.html)


## Components

### Data Down Action Up
If we want to create app which embraces immutable data structures, we have to use DDAU convention. In a nutshell you should't change passed data in components instead trigger actions that should change this data.

```hbs
// GOOD
{{nice-component currentData=(action 'freshData')}}
```
```javascript
export default Component.extend({
	actions: {
    	operationX() {
        	this.attr.currentData();
        }
    }
});

export default Controller.extend({
	actions: {
    	freshData() {
        	return sophisticateComputation();
        }
    }
});
```
```hbs
// BAD
{{uggly-component currentData=freshData}}
```
```javascript
export default Component.extend({
	actions: {
    	operationY() {
        	this.set('currentData', 'cetasean');
        }
    }
});
```

## Tests

### Use page objects in acceptance testing
In acceptance tests there is a tendency to repeat the same code meny times (mainly selectors). To omit this we can extract this to other abstract layer and then import them to tests.

```javascript
// GOOD
export default Ember.Object({
	assertMessage(msg) {
      andThen(() => {
        this.get('assert').equal(find('#message').text(), msg);
      });
      return this;
    }
})
-----
import PageObject from 'my-project/tests/page-objects/base';
...
test('check changed message', function(assert) {
	PageObject.create({ assert })
    		  .assertMessage('cabbage');
});
```
