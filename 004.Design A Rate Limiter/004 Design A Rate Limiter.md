# 004.Design A Rate Limiter

- In a network system, a rate limiter is a device to control the processing rate of traffic sent by a client or service.
- Taking HTTP as an example, it limits the number of client requests sent within a certain period of time.
- If the number of API requests exceeds the threshold, processing of additional calls will be stopped.

## Example

- Users cannot post new posts more than twice per second
- Users cannot create more than 10 accounts per day with the same IP address

## Benefits of Rate Limiter

- Can prevent resource depletion caused by DoS (Denial of Service) attacks
- Reduce costs. Limiting processing means you don't have to have as many servers, which is very important for companies paying for third-party APIs.
- Prevent server overload. Traffic from bots or traffic caused by incorrect usage patterns can be filtered out.

## Step 1: Understand the problem and confirm the design scope
Communicating with the interviewer can make it clear what restrictions need to be implemented.

- Is it a client side limiter or a server side limiter?
- What are the criteria that control API calls? IP addresses? Or user ID?
- Does the system need to operate in a distributed environment?

## Step 2: Present a rough design and obtain consent

Where to place the Rate Limiter?

**1) Client side:** 

client requests can be easily forged

**2) Server side:**

    **2-1) API server**

![Untitled](./images/Untitled.png)

    **2-2) Middleware**

![Untitled](./images/Untitled%201.png)

- Rate limiting middleware: API gateway
- In the case of cloud microservices, the rate limiter is usually implemented in the API gateway.
- API Gateway is a fully entrusted management service that supports processing rate limiting, SSL termination, user authentication, and IP whitelist management.

## Processing rate limiting algorithm

**1) Token bucket**
Tokens are periodically filled in a container (bucket) with a specified capacity, and each request is processed using one token each time it is processed. If there is no token, the request is discarded.

**2) Leaky bucket**
Implemented as a FIFO queue, it checks whether the queue is full and adds a request to the queue if there is an empty space. If the queue is full, new requests are discarded. Finally, requests are pulled from the queue at specified times and processed.

**3) Fixed window counter**
Divide the timeline into windows at fixed intervals and add a counter to each window. Increments the count for each request based on the timestamp (minute, hour, etc.). If the count exceeds the threshold, the request is discarded.

**4) Sliding window log**

- New requests add a timestamp to the log. If the log size is equal to or smaller than the allowable value, the request is forwarded to the system, otherwise, processing is refused.
- When a new request comes, the expired timestamp is removed, and an expired timestamp means that its value is older than the start time of the current window.

**5) Sliding window counter**

- It is a combination of the fixed window counter algorithm and the sliding window logging algorithm.
- Formula = Number of requests in the current 1 minute + Number of requests in the previous 1 minute x Rate of overlap between the moving window and the previous 1 minute
- Processes a request as large as the value of the above formula in the current window.

## Rough architecture

- The rate limiter algorithm has a counter that can track how many requests have been received, and if the value of this counter exceeds a certain limit, requests that arrive beyond the limit are rejected.
- It is desirable to store counters in a cache that runs in memory (supports fast, time-based expiration policy)
- Example: Redis Rate Limiter

**1) Command**
- INCR: Increases the counter value by 1
- EXPIRE: Set a timeout value for the counter, and the counter is automatically deleted when the set time elapses.

**2) Principle of operation**
- Client sends request to rate limiting middleware
- The middleware retrieves a counter from a designated bucket in Redis and checks whether the limit has been reached.
- If the limit is reached, the request is rejected
- If the limit has not been reached, the request is forwarded to the API server.
- The middleware increases the counter value and stores it back in Redis.

## Step 3: Detailed Design

- Processing rate limit rules are stored on disk in the form of a configuration file.
- Example: Restrict clients from logging in more than 5 times per minute

 domain: auth
descriptors:
       - key: auth_type
         Value: login
         rate_limit:
                  unit: minute
                  requests_per_unit: 5

## Processing of traffic exceeding rate limit

- If a request hits the limit, the API sends an HTTP 429 response (too many requests) to the client.
- The client can tell whether its request is subject to the rate limiter through the HTTP response header.
X-Ratelimit-Remaining: Number of requests remaining within the window.
X-Ratelimit-Limit: Number of requests a client can send per window
X-Ratelimit-Retry-After: tells you how many seconds to resend the request to avoid hitting the limit

## Detailed design

- Rate Limiter rules are stored on disk.
- Work process (workers) frequently read rules from disk and store them in the cache.
- When a client sends a request to the server, the request first reaches the rate limiting middleware.
- Rate limiting middleware retrieves limiting rules from cache, counter and timestamp of last request from Redis cache.
- Based on the retrieved values, the middleware makes the following decisions:
ㆍ If the request is not subject to rate limiters, it is sent to the API server.
ㆍ If the limit is reached, a 429 too many requests error is sent to the client.
ㆍ The request is discarded or stored in the message queue.

## Implementation of Rate Limiter in a distributed environment

Although easy to implement on a single server, scaling the system to support multiple servers and parallel threads requires addressing two issues

**1) Race conditions**

- If threads processing two requests read the counter value in parallel, both will write the value of counter plus 1 to Redis, regardless of the processing status of other requests.
- The solution to this is lock. However, since locks significantly reduce system performance, it can be better to use a Lua script or Redis' sorted set.

![Untitled](./images/Untitled%202.png)

**2) Synchronization issue**

- If you have multiple Rate Limiter servers, synchronization becomes necessary.
- Because the web layer is stateless, clients can send requests to different limiting devices.
- Device 1 knows nothing about Client 2 and therefore cannot properly perform rate limiting.
- The solution is to utilize sticky sessions and always send requests from the same client to the same Rate Limiter.

![Untitled](./images/Untitled%203.png)

However, since it is difficult to scale and inflexible, a better solution is a centralized data store such as Redis.

![Untitled](./images/Untitled%204.png)

## Performance optimization

1. **Support for multiple data centers**
- Users located far from the data center may experience increased processing latency.
- Cloud service providers plant edge servers around the world and forward user traffic to the nearest edge server to reduce latency.

1. **Eventual consistency model**
When synchronizing data between devices, use an eventual consistency model (data consistency)

## Monitoring

- We need to collect data to see if the Rate Limiter is working effectively.
- If the rules are set too tightly, many valid requests may be discarded without being processed.
- When traffic surges due to events such as flash sales, the algorithm must be changed efficiently (token buckets are suitable).

## Step 4: Finishing

- Hard throttling: the number of requests can never exceed the threshold.
- Soft throttling: the number of requests can exceed the threshold for a while
- How to design clients to avoid throughput limitations
ㆍ Reduce the number of API calls using client-side cache
ㆍ Avoid sending too many messages in a short period of time.
ㆍ Handles exceptions and errors so they can recover gracefully.
ㆍ When implementing retry logic, allow sufficient back-off time.