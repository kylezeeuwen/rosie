# Rosie

[![Build Status](https://travis-ci.org/rosiejs/rosie.svg?branch=master)](https://travis-ci.org/rosiejs/rosie)

![Rosie the Riveter](http://upload.wikimedia.org/wikipedia/commons/thumb/1/12/We_Can_Do_It%21.jpg/220px-We_Can_Do_It%21.jpg)

Rosie is a factory for building JavaScript objects, mostly useful for setting up test data. It is inspired by [factory_girl](https://github.com/thoughtbot/factory_girl).

To use Rosie you first define a _factory_. The _factory_ is defined in terms of _attributes_, _sequences_, _options_, _callbacks_, and can inherit from other factories. Once the factory is defined you use it to build _objects_.

## Usage

There are two phases of use:

1. Factory definition
2. Object building

**Factory Definition:** Define your factory, giving it a name and optionally a constructor function (`game` in this example):

```js
Factory.define('game')
  .sequence('id')
  .attr('is_over', false)
  .attr('created_at', function() { return new Date(); })
  .attr('random_seed', function() { return Math.random(); })

  // Default to two players. If players were given, fill in
  // whatever attributes might be missing.
  .attr('players', ['players'], function(players) {
    if (!players) { players = [{}, {}]; }
    return players.map(function(data) {
      return Factory.attributes('player', data);
    });
  });

Factory.define('player')
  .sequence('id')
  .sequence('name', function(i) { return 'player' + i; })

  // Define `position` to depend on `id`.
  .attr('position', ['id'], function(id) {
    var positions = ['pitcher', '1st base', '2nd base', '3rd base'];
    return positions[id % positions.length];
  });

Factory.define('disabled-player').extend('player').attr('state', 'disabled')
```

**Object Building:** Build an object, passing in attributes that you want to override:

```js
var game = Factory.build('game', {is_over:true});
// Built object (note scores are random):
//{
//    id:           1,
//    is_over:      true,   // overriden when building
//    created_at:   Fri Apr 15 2011 12:02:25 GMT-0400 (EDT),
//    random_seed:  0.8999513240996748,
//    players: [
//                {id: 1, name:'Player 1'},
//                {id: 2, name:'Player 2'}
//    ]
//}
```

For a factory with a constructor, if you want just the attributes:

```js
Factory.attributes('game') // return just the attributes
```

### Programmatic Generation of Attributes

You can specify options that are used to programatically generate the attributes:

```js
var moment = require('moment');

Factory.define('matches')
  .attr('seasonStart', '2016-01-01')
  .option('numMatches', 2)
  .attr('matches', ['numMatches', 'seasonStart'], function(numMatches, seasonStart) {
    var matches = [];
    for (var i = 1; i <= numMatches; i++) {
      matches.push({
        matchDate: moment(seasonStart).add(i, 'week').format('YYYY-MM-DD'),
        homeScore: Math.floor(Math.random() * 5),
        awayScore: Math.floor(Math.random() * 5)
      })
    }
    return matches;
  })

Factory.build('matches', { seasonStart: '2016-03-12' }, { numMatches: 3 })
// Built object (note scores are random):
//{
//  seasonStart: '2016-03-12',
//  matches: [
//    { matchDate: '2016-03-19', homeScore: 3, awayScore: 1 },
//    { matchDate: '2016-03-26', homeScore: 0, awayScore: 4 },
//    { matchDate: '2016-04-02', homeScore: 1, awayScore: 0 }
//  ]
//}
```

In the example `numMatches` is defined as an `option`, not as an `attribute`. Therefore `numMatches` is not part of the output, it is only used to generate the `matches` array.

In the same example `seasonStart` is defined as an `attribute`, therefore it appears in the output, and can also be used in the generator function that creates the `matches` array.

### Post Build Callback

You can also define a callback function to be run after building an object:

```js
Factory.define('coach')
  .option('buildPlayer', false)
  .sequence('id')
  .attr('players', ['id', 'buildPlayer'], function(id, buildPlayer) {
    if (buildPlayer) {
      return [Factory.build('player', {coach_id: id})];
    }
  })
  .after(function(coach, options) {
    if (options.buildPlayer) {
      console.log('built player:', coach.players[0]);
    }
  });

Factory.build('coach', {}, {buildPlayer: true});
```

Multiple callbacks can be registered, and they will be executed in the order they are registered. The callbacks can manipulate the built object before it is returned to the callee.

### Associate a Factory with an existing Class

This is an advanced use case that you can probably happily ignore, but store this away in case you need it.

When you define a factory you can optionally provide a class definition, and anything built by the factory will be passed through the constructor of the provided class.

Specifically, the output of `.build` is used as the input to the constructor function, so the returned object is an instance of the specified class:

```js
var SimpleClass = function(args) {
  this.moops = 'correct';
  this.args = args
};

SimpleClass.prototype = {
  isMoopsCorrect: function() {
    return this.moops;
  }
}

testFactory = Factory.define('test', SimpleClass)
  .attr('some_var', 4)

testInstance = testFactory.build({ stuff: 2 });
console.log(JSON.stringify(testInstance, {}, 2))
// Output:
// {
//   "moops": "correct",
//   "args": {
//     "stuff": 2,
//     "some_var": 4
//   }
// }

console.log(testInstance.isMoopsCorrect());
// Output:
// correct

```

Mind. Blown.

## Usage in Node.js

To use Rosie in node, you'll need to require it first:

```js
var Factory = require('rosie').Factory;
```

You might also choose to use unregistered factories, as it fits better with node's module pattern:

```js
// factories/game.js
var Factory = require('rosie').Factory;

module.exports = new Factory()
  .sequence('id')
  .attr('is_over', false)
  // etc
```

To use the unregistered `Game` factory defined above:

```js
var Game = require('./factories/game');

var game = Game.build({is_over: true});
```

## Usage in ES6

Unregistered factories are even more natural in ES6:

```js
// factories/game.js
import { Factory } from 'rosie';

export default new Factory()
  .sequence('id')
  .attr('is_over', false)
  // etc

// index.js
import Game from './factories/game');

const game = Game.build({is_over: true});
```

A tool like [babel](https://babeljs.io) is currently required to use this syntax.

## Rosie API

As stated above the rosie factory signatures can be broken into factory definition and object creation.

Additionally factories can be defined and accessed via the Factory singleton, or they can be created and maintained by the callee.

###Factory declaration functions

Once you have an instance returned from a `Factory.define` call, you do the actual of work of defining the objects. This is done the methods below on the Factory instance that is returned from `Factory.define` or from `new Factory()`  (note these are typically chained together as in the examples above):

#### Factory.define

* **Factory.define(``factory_name``)** - Defines a factory by name. Return an instance of a Factory that you call `.attr`, `.option`, `.sequence`, and `.after` on the result to define the properties of this factory.
* **Factory.define(`factory_name`, `constructor`)** - Optionally pass a constuctor function, and the objects produced by `.build` will be passed through the `constructor` funtion.


#### instance.attr:

Use this to define attributes of your objects

* **instance.attr(`attribute_name`, `default_value`)** - `attribute_name` is required and is a string, `default_value` is the value to use by default for the attribute
* **instance.attr(`attribute_name`, `generator_function`)** - `generator_function` is called to generate the value of the attribute
* **instance.attr(`attribute_name`, `dependencies`, `generator_function`)** - `dependencies` is an array of strings, each string is the name of an attribute or option that is required by the `generator_function` to generate the value of the attribute. This string of `dependencies` will match the parameters that are passed to the `generator_function`

#### instance.option:

Use this to define options used to programatically generate attributes of your objects

* **instance.option(option_name, `default_value`)** - option_name is required and is a string, `default_value` is the value to use by default for the option
* **instance.option(option_name, `generator_function`)** - `generator_function` is called to generate the value of the option
* **instance.option(option_name, `dependencies`, `generator_function`)** - `dependencies` is an array of strings, each string is the name of an attribute or option that is required by the `generator_function` to generate the value of the option. This string of `dependencies` will match the parameters that are passed to the `generator_function`

#### instance.sequence:

Use this to define an auto incrementing sequence field in your object

* **instance.sequence(`sequence_name`)** - define a sequence called `sequence_name`, set the start value to 1
* **instance.sequence(`sequence_name`, `generator_function`)** - `generator_function` is called to generate the value of the sequence
* **instance.sequence(`sequence_name`, `dependencies`, `generator_function`)** - `dependencies` is an array of strings, each string is the name of an attribute or option that is required by the `generator_function` to generate the value of the option. This string of `dependencies` will match the parameters that are passed to the `generator_function`

#### instance.after:

* **instance.after(`callback`)** - register a `callback` function that will be called at the end of the object build process. The `callback` is invoked with two params: (`build_object`, `object_options`)

###Object building functions

#### build

Returns an object that is generated by the named factory. `attributes` and `options` are optional parameters. The `factory_name` is required when calling against the rosie Factory singleton.

* **Factory.build(`factory_name`, `attributes`, `options`)** - when build is called against the rosie Factory singleton, the first param is the name of the factory to use to build the object. The second is an object containing attribute override key value pairs, and the third is a object containing option key value pairs
* **instance.build(`attributes`, `options`)** - when build is called on a factory instance only the `attributes` and `options` objects are necessary

#### buildList

Identical to `.build` except it returns an array of built objects. `size` is required, `attributes` and `options` are optional

* **Factory.buildList(`factory_name`, size, `attributes`, `options`)** - when buildList is called against the rosie Factory singleton, the first param is the name of the factory to use to build the object. The `attributes` and `options` behave the same as the call to `.build`.
* **instance.buildList(size, `attributes`, `options`)** - when buildList is called on a factory instance only the size, `attributes` and `options` objects are necessary (strictly speaking only the size is necessary)

## Contributing

0. Fork it
0. Create your feature branch (`git checkout -b my-new-feature`)
0. Install the test dependencies (`script/bootstrap` - requires NodeJS and npm)
0. Make your changes and make sure the tests pass (`npm test`)
0. Commit your changes (`git commit -am 'Added some feature'`)
0. Push to the branch (`git push origin my-new-feature`)
0. Create new Pull Request

## Credits

Thanks to [Daniel Morrison](http://twitter.com/danielmorrison/status/58883772040486912) for the name and [Jon Hoyt](http://twitter.com/jonmagic) for inspiration and brainstorming the idea.
