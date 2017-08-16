---
layout: default
title: Express.js A/B Testing
category: nodejs
tags:
- nodejs
- expressjs
---
![Node Express AB Testing]({{ site.url }}/assets/node_express_ab_testing.jpg)
A few years ago, I was exploring ways to perform simple multivariate tests in my node.js web application. Working in an express.js environment, it was a natural conclusion to render a different page for each variant, and to use middleware for handling the traffic splitting. In this post, we'll explore how to implement this routine in your own express.js application.

Let's begin with our basic express application.

```javascript
const express = require('express');

const PORT = 3000;
const app = express();
```

Next, let's register two route handlers. Each will render a unique view for the same route depending on our A/B test middleware. For this example, we'll simply return a JSON response for both our variants. In your app, you would most likely swap `res.send` for `res.render`, and specify a different template file for each.

```javascript
app.get('/deals', function dealsA(req, res) {
  res.send({ variant: 'a' });
});

app.get('/deals', function dealsB(req, res) {
  res.send({ variant: 'b' });
});
```

If we added the necessary code to start our application server, and loaded it in the browser, we would never see the `deals-b` response. In express, route handlers are executed in the order they are registered, hence our second handler is never reached. To get around this, we'll implement express middleware and make use express' `next()` function. All route handling functions in express receive three arguments: `request`, `response`, and `next`. This article assumes familiarity with express.js, and the arguments mentioned. If you aren't familiar, or need to refresh your memory, give the [Express.js routing documentation](https://expressjs.com/en/starter/basic-routing.html) a read.

The most common use of `next` is to pass control down to error handlers. Calling `next()` without arguments to trigger our 404 handler, or `next(error)` to execute our error handler. A lesser-known use of `next` is with the `'route'` argument. The [documentation](https://expressjs.com/en/guide/using-middleware.html) explains that calling `next('route') instructs express to execute the subsequent matching route handler. We'll make use of this functionality to build our middlware.

### A/B Test Middleware
Our package will expose a factory function `ab()`, which returns a function to generate a variant for each of our routes. Inside our test instance, we keep a dictionary of our test variants; storing the number of times we've executed that specific route handler. Lastly, the middleware returned for each test variant determines if it should execute its handler, or continue down the routing chain. Here, we'll opt  for a dead simple weighting system -- an even spread across all variants. In an actual distributed production environment, you should consider more robust logic.

```javascript
// ab.js
function ab() {
  // store a counter for each variant
  const counter = {};
  // return function to register variant
  return function test(variant) {
    counter[variant] = 0;
    // return express middleware
    return function (req, res, next) {
      // check if variant has fewest requests
      // if not, next('route') to go to next variant
      // otherwise, increment variant counter and next()
      const current = counter[variant];
      const smallest = !Object.values(counter).some(count => count < current);

      if (!smallest) return next('route');

      counter[variant] = current + 1;
      next();
    };
  };
}

module.exports = ab;
````
That's all there is to it. Now let's see how we can integrate it into our express.js application.
```
const ab = require('./ab');

// Create our test instance
const dealsABTest = ab();

// Register a variant for each controller. Function returns an express middleware
app.get('/deals', dealsABTest('variantA'), function dealsA(req, res) {
  res.send({ variant: 'a' });
});

app.get('/deals', dealsABTest('variantB'), function dealsB(req, res) {
  res.send({ variant: 'b' });
});
```

Given our simple weighting logic, we can expect an even split of traffic between our two route handlers. All that's left is to start our express server. Add the following code to the end of your app file.

```
app.listen(3000, function () {
  console.log('Express application listening on port 3000');
});
```

To recap, the routine outlined above enables simple multivariate testing in express.js applications. Test variants are defined as unique route handlers, and are routed to evenly by our express middleware. I've published this middleware to NPM as [abn](https://github.com/NoumanSaleem/abn). The package provides a crucial component left out of this article -- ensuring the user is presented the same variant on subsequent requests.
