# retry

Abstraction for exponential and custom retry strategies for failed operations.

## Current Status

At this point I'm just writing the Readme to get a feel for the API. Once I
like it, I'll implement it.

## Tutorial

The example below will retry a potentially failing `dns.resolve` operation
`10` times using an exponential backoff strategy. With the default settings, this
means the last attempt is made between `34 minutes and 7 seconds` and `1 hour
8 minutes and 14 seconds`. Disable `randomize` setting if you want a
deterministic timeout sequence.

``` javascript
var dns = require('dns');
var retry = require('retry');

function faultTolerantResolve(address, cb) {
  var operation = retry.operation();

  operation.try(function() {
    dns.resolve(address, function(err, addresses) {
      if (operation.retry(err)) {
        return;
      }

      cb(operation.mainError, addresses);
    });
  });
}

faultTolerantResolve('nodejs.org', function(err, addresses) {
  console.log(err, addresses);
});
```

Of course you can also configure the factors that go into the exponential
backoff. See the API documentation below for all available settings.

``` javascript
var operation = retry.operation({
  times: 5,
  factor: 3,
  minTimeout: 1 * 1000,
  maxTimeout: 60 * 1000,
  randomize: false,
});
```

## API

### retry.operation([options])

Shortcut for `new RetryOperation(options)`.

### new RetryOperation([options])

Creates a new instance of the `RetryOperation` class. All time related values
are given in milliseconds.

`options` is a JS object that can contain any of the following keys:

* `times`: The maximum amount of times to retry the operation. Default is `10`.
* `factor`: The exponential factor to use. Default is `2`.
* `minTimeout`: The minium amount of time between two retries. Usually only applies to the first retry. Default is `1000`.
* `maxTimeout`: The maximum amount of time between two retries. Default is `Infinity`.
* `randomize`: Randomizes the timeouts by multiplying with a factor between `1` to `2`. Default is `true`.

The formula used to calculate the individual timeouts is:

```
var timeout = Math.min(random * minTimeout * Math.pow(factor, attempt), maxTimeout);
```

Have a look at [this article][article] for a better explanation of approach.

If you want to tune your `factor` / `times` settings to attempt the last retry
after a certain amount of time, you can use wolfram alpha. For example in order
to tune for `10` attempts in `5 minutes`, you can use this
query:

[http://www.wolframalpha.com/input/?i=Sum%5Bx^k%2C+{k%2C+0%2C+10}%5D+%3D+5+*+60]()

[article]: http://dthain.blogspot.com/2009/02/exponential-backoff-in-distributed.html

## retryOperation.try(fn)

Defines the function `fn` that is to be retried and executes it for the first
time right away.

## retryOperation.retry(error)

Returns `false` when no `error` value is given, or the maximum amount of retries
has been reached.

Otherwise it executes the operation again, and returns `true`. It also typecasts
the `error` value into an `Error` object as neccessary, and adds it to the
list of `retryOperation.errors`. This list is kept to determine the
`retryOperation.mainError`.