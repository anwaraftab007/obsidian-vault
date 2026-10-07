## Authentication Architectures
1. **Stateful (Sessions):** 
   - The server stores user session data in a database (e.g., Redis) and sends a random Session ID cookie to the client.
   - **Pros/Cons:** Highly secure and easily revocable. Slower because it requires a database lookup on every single request.
   - **Crucial Rule:** Never store sessions in server RAM (`req.session`) when using clustering (PM2), as memory is not shared across CPU cores.

2. **Stateless (JSON Web Tokens - JWT):**
   - The server creates a token containing the user's ID, signs it cryptographically, and sends it to the client.
   - **Pros/Cons:** Extremely fast (no database lookups required to verify identity). Hard to revoke before the token naturally expires.

## Anatomy of a JWT (`Header.Payload.Signature`)
- **Header:** Declares the hashing algorithm used.
- **Payload:** Contains the actual data (e.g., `userId`). **WARNING:** This is only Base64 encoded, NOT encrypted. Never put sensitive data (passwords, PII, credit cards) in the payload.
- **Signature:** A cryptographic hash of the Header + Payload + Server Secret Key. If anyone alters the payload, the signature verification will fail, preventing tampering.

### Essential Express Security Middleware

Besides authentication, you must protect your Express app from common web attacks:

- **Helmet:** A middleware that automatically sets HTTP response headers to protect against Cross-Site Scripting (XSS) and clickjacking. (`app.use(helmet())`)
    
- **CORS (Cross-Origin Resource Sharing):** By default, browsers block web pages from making requests to a different domain. CORS middleware allows you to specify exactly which frontend URLs are allowed to talk to your backend.
    
- **Rate Limiting:** Protects against DDoS attacks and brute-force password guessing by limiting how many requests a single IP address can make in a given timeframe (e.g., 100 requests per 15 minutes).


![[Pasted image 20261004101640.png]]


### Knowledge Check

Reply with your answers to complete the final phase!

1. **You are building an Express API using JWTs. You decide to put the user's home address and credit card number inside the JWT Payload so you don't have to query the database later. Why is this a terrible security flaw?**
    
2. **If you use Stateful Session authentication, and you store the session data in server memory (`req.session`) instead of an external database like Redis, what happens to your users when you deploy your app using PM2 Clustering?** _(Hint: Think about your answer to question 1 from Phase 4)_

**1. Unencrypted JWT Payload** You are exactly right that the payload is not encrypted—it is merely Base64 encoded, meaning anyone who intercepts the token can read the credit card number in plain text. _Minor clarification on tampering:_ While a hacker _can_ intercept and alter the payload, the server will reject it. The moment the payload is changed, the server's cryptographic signature check will fail. The danger here is purely about **data exposure**, not data tampering.

**2. In-Memory Sessions with PM2** Spot on. This is the exact memory isolation issue from Phase 4. If a user logs in and Core 1 saves their session in memory, their very next HTTP request might be routed by PM2 to Core 2. Core 2 has a blank memory state, assumes the user is an intruder, and throws a 401 Unauthorized. This is exactly why distributed backends mandate external session stores like Redis.


[[BACKEND ENGINEERING]]