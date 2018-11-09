# bookshelf-validate

[![Version (npm)](https://img.shields.io/npm/v/bookshelf-validate.svg)](https://npmjs.com/package/bookshelf-validate)

Validation for the Model objects of Bookshelf.js

## Installation

```
npm install --save bookshelf-validate
```

## Configuration

To initialize the plugin, call `bookshelf.plugin('bookshelf-validate'[, config]);`
with your Bookshelf instance.

The `config` object is optional, and the defaults are:

```js
{
  validator: require('validator'), // node-validator
  validateOnSave: false // Automatically validate when Bookshelf emits 'saving' event
}
```

### config.validator

By default, the validation methods are provided by the __validator__ npm package.

You can pass a different validator object, or customize node-validator by adding
methods to it before passing it to the configuration. The only requirement
is that every method on the validator object takes the value to be validated as the
first argument.

Example by customizing __validator__:

``` js
var validator = require('validator');

validator.isPrime = function (str) {
  var value = parseInt(str);
  if (value === NaN || value < 2) return false;

  for (var i = 2; i <= value >> 1; i++) {
    if (value % i === 0) {
      return false;
    }
  }
};

bookshelf.plugin('bookshelf-validate', {
  validator: validator
});
```

### config.validateOnSave

If `validateOnSave` is `true`, the validations will automatically be called
when Bookshelf emits a `'saving'` event. If any validations fail, an
error will be thrown and the model will not be saved.

## API

### 'isRequired'

Since `node-validator` does not look at undefined or null values, an optional
validator `isRequired` is provided with the library. It is off by default.

### Model.validations

To use `bookshelf-validator` in your models, you must add a `validations`
key to your Bookshelf model. Each key of the `validations` object must be
an attribute of the Bookshelf model. There are four ways to write the
validations for each attribute, depending on the level of specifity you need
to provide.

#### Option 1: Validation method name as string

```js
var User = bookshelf.Model.extend({
  validations: {
    username: 'isRequired'
  }
});
```

#### Option 2: Validation method name as key in object

With this option, the values of the validation method name keys will be passed
to the validation methods as arguments. An array of values may be passed
for arbitrary-arity methods.

```js
var User = bookshelf.Model.extend({
  validations: {
    username: {
      isRequired: true,
      isLength: { min: 2, max: 32 } // will call validator.isLength(value, { min: 2, max: 32 })
    }
  }
});
```


#### Option 3: Validation method with custom error message

With this option, if the validation fails, your custom error message will be
returned instead of the default (the name of the method which failed validation).
This is good if you want to send messages to the user about why validation failed.

```js
var User = bookshelf.Model.extend({
  validations: {
    username: {
      method: 'isLength',
      error: 'Your username must be between 4 and 32 characters long.',
      args: { min: 4, max: 32 }
    }
  }
});
```

The `error` and `args` attributes are optional.

#### Option 4: An array with a combination of any of the above

For more complicated validations, you may pass an array that has a combination
of strings and objects. If an object has an attribute called `method` it will be
assumed to be type (3), otherwise it will be type (2). Since you will probably
not name a validation method with the name `method`, it is unlikely there will
be any problems.

```js
var User = bookshelf.Model.extend({
  validations: {
    email: [
      'isRequired',
      { isEmail: {allow_display_name: true} }, // Options object passed to node-validator
      { method: 'isLength', error: 'Username 4-32 characters long.', args: { min: 4, max: 32 } } // Custom error message
    ]
  }
});
```

### Model#validationErrors

Call `validationErrors()` on a model instance to see the results of your validation.

If there are any invalid attributes on your model, the result will be a
`ValidationError` object containing an object of attributes and their errors on
the `data` property.

If the `validateOnSave: true` option was configured, `validationErrors()` will
be called automatically when you attempt to save the model. If not, you can
call it yourself to see the errors before you save the model.

``` js
var bookshelf = require('bookshelf');
var validator = require('validator');

validator.isRequired = function (val) {
  return val != null;
}

bookshelf.plugin('bookshelf-validate', {
  validator: validator,
  validateOnSave: true
});

var User = bookshelf.Model.extend({
  tableName: 'users',
  validations: {
    // Username is required, and its length must be between 2 and 32 characters
    username: [
      'isRequired',
      { method: 'isLength', error: 'Username must be between 2 and 32 characters.', args: { min: 2, max: 32 } }
    ],

    // Email is required, and must be a valid e-mail address
    email: [
      'isRequired',
      { method: 'isEmail', error: 'Not a valid email address' }
    ],

    // Birthday is not required, but must be a date if given
    birthday: { isDate: true } // Same as `birthday: 'isRequired'`
  }
});

var user = new User({
  username: 'x', // Invalid length
  birthday: '1997-11-21', // Valid
  color: 'green' // Not validated
});

let errors = user.validationErrors();
console.log(errors);
/*
  [ValidationError]:
  {
    data: {
      username: ['Username must be between 2 and 32 characters'],
      email: ['isRequired']
    }
  }
/*
```

And, if `validateOnSave` is `true`:

```js
user.save(); // Throws a ValidationError
```
