# Core requirements
- Reverse proxy and load balancer (TLS termination)
- Authentication
- URL Shortening
- Redirection
- Analytics
- Caching (Redis)
- Message Queue (Kafka)
- Persistent data storage (Database)

## Reverse Proxy (Nginx + OpenResty)
Responsible for decrypting HTTPS traffic, validating JWT (for specific routes) and forwarding (load balance) traffic to backend services as unencrypted HTTP along with rate limiting of requests. Also, it authenticates each request and upon successful authentication, route traffic to healthy service based on endpoint.

- Direct Auth Endpoint (/auth/login, /auth/register):
  - Client -> Nginx (TLS termination) -> Auth Service -> Client
  - These bypasses the token-validation middleware because the user is actively trying to obtain JWT
- Protected Endpoint (/shorten, /analytics/{short_code})
  - Step A: Client -> Nginx(with Authorization: Bearer <token>)
  - Step B (Internal validation using OpenResty): Lua script is used to validate JWT.
  - Step C: Upon nginx validating JWT, the request is routed to respective service.

## Authentication
Responsible for registering or logging in users and issuing JWT. It creates user details in the database.

## URL Shortening
When a user submits a long URL, we need to generate a unique short code, store the mapping and return the short URL. It handles all write operations. It validates incoming URLs, handles custom alias the user provides and generate unique short code along with storing expiry date, creation time and userID. Since write traffic is relatively low compared to read traffic, it doesn't need to scale aggressively.

## URL Redirection
It handles majority of the traffic, typically billions of GET requests per day, so it must be fast, stateless and scales horizontally. For a given short code, it looks up the original long URL, checks if the URL is still valid (not expires or deleted), and issues an HTTP 301 (permanent) or 302 (temporary) redirect based on system's policy. It also caches frequently accessed mappings to minimize database hits.

There is also a worker pool that pulls job from a channel. On this channel, the short_url is provided that is then published on a topic on kafka by one of the worker.

## Analytics
Analytics service is responsible for reading from kafka queue on fixed interval (5s) and aggregate result and then updating database accordingly. It also provides click count to verified users for their specified short_url.