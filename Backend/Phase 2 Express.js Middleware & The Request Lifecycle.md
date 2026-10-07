## The Middleware Chain
Express is a routing and middleware web framework. An application is essentially a sequential array of functions.
- **The Golden Rule:** Every middleware must either end the request-response cycle (e.g., `res.json()`) or pass control to the next function using `next()`. If neither happens, the request hangs until it times out.
- **Top-to-Bottom Execution:** Express iterates through registered middlewares in the exact order they are declared. State matters (e.g., body parsers must be declared *before* the routes that need to read `req.body`).

## Middleware Types
1. **Application-level:** Bound to `app.use()`. Runs globally (e.g., CORS, logging).
2. **Router-level:** Bound to `express.Router()`. Allows modularizing routes (e.g., separating `/users` from `/products`).
3. **Route-handling:** The final stop, usually executing business logic and returning a response.

## The Error Handling Bypass (Critical)
- Standard middleware takes 3 arguments: `(req, res, next)`.
- **Error-handling middleware takes 4 arguments:** `(err, req, res, next)`.
- **The Bypass Mechanism:** If you call `next(err)` inside any standard route or middleware, Express immediately aborts the normal chain. It skips all remaining standard middlewares and jumps down the file to the first 4-argument error handler it finds.
- **Placement:** Because of this downward execution, global error handlers MUST be the very last `app.use()` statement in your application.

When a request hits your server, Express iterates through the array of registered middlewares in the **exact order they were declared** (top to bottom in your code).

- **The Golden Rule:** Every single middleware MUST either end the request-response cycle (e.g., `res.send()`, `res.json()`) OR pass control to the next function by calling `next()`. If a middleware does neither, the client's request will hang until it times out.
    
- **Order dictates state:** If you place `app.use(express.json())` (which parses JSON bodies) _after_ your route handler, your route will see `req.body` as `undefined`.


[[BACKEND ENGINEERING]]