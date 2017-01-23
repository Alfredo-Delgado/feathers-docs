# Express middleware

In Express [middleware functions](http://expressjs.com/en/guide/writing-middleware.html) are functions that have access to the request object (`req`), the response object (`res`), and the next middleware function in the application’s request-response cycle. The next middleware function is commonly denoted by a variable named `next`. How this middleware plays with Feathers [services](../services/readme.md) is outlined below.

## Rendering views

While services primarily provide APIs for a client side application to use, they also play well with [rendering views on the server with Express](http://expressjs.com/en/guide/using-template-engines.html). For more details, please refer to the [Using a View Engine](../guides/using-a-view-engine.md) guide.

## Custom service middleware

Custom Express middleware that only should run before or after a specific service can be passed to `app.use` in the order it should run:

```js
const todoService = {
  get(id) {
    return Promise.resolve({
      id,
      description: `You have to do ${id}!`
    });
  }
};

app.use('/todos', ensureAuthenticated, logRequest, todoService, updateData);
```

Middleware that runs after the service will have `res.data` available which is the data returned by the service. For example `updateData` could look like this:

```js
function updateData(req, res, next) {
  res.data.updateData = true;
  next();
}
```

Keep in mind that shared authentication (between REST and websockets) should use a service based approach as described in the [authentication guide](../authentication/readme.md).

Information about how to use a custom formatter (e.g. to send something other than JSON) can be found in the [REST provider](../rest/readme.md) chapter.

## Setting service parameters

All middleware registered after the REST provider has been configured will have access to the `req.feathers` object to set properties on the service method `params`:

```js
app.configure(rest())
  .use(bodyParser.json())
  .use(bodyParser.urlencoded({extended: true}))
  .use(function(req, res, next) {
    req.feathers.fromMiddleware = 'Hello world';
    next();
  });

app.use('/todos', {
  get(id, params) {
    console.log(params.provider); // -> 'rest'
    console.log(params.fromMiddleware); // -> 'Hello world'

    return Promise.resolve({
      id, params,
      description: `You have to do ${id}!`
    });
  }
});
```

We recommend not setting `req.feathers = something` directly since it may already contain information that other Feathers plugins rely on. Adding individual properties or using `Object.assign(req.feathers, something)` is the more reliable option.

> __ProTip:__ Although it may be convenient to set `req.feathers.req = req;` to have access to the request object in the service, we recommend keeping your services as provider independent as possible. There usually is a way to pre-process your data in a middleware so that the service does not need to know about the HTTP request or response.
