# From Errors to Elegance: Handling Redis Rate Limiting and Database Connections in PHP

Working on high-availability PHP applications always presents one truth: every dependency you introduce can fail. 

---

## The Initial Setup

My application collects and serves data using a MySQL database. To avoid abuse and excessive traffic, I implemented a rate limiter using Redis.

In PHP, I was using a custom `RateLimiter` class that connected to Redis and tracked IP requests. Everything worked beautifully — **until it didn’t**.

I didn't take into consideration that if the RateLimiter service would go down or not start up properly, 
that it would mean that it's not possible to get a connection to the database which is okay(?) but not necessarily what I want for this use case
---

## The Problem

Once in production, I started getting a **critical error** message:

A critical error occurred. Please try again later.
Fatal error: Uncaught Error: Call to a member function prepare() on null


This wasn’t a database issue per se — it was a cascading failure caused by Redis. The `RateLimiter` failed to connect, 
but that failure bubbled up and **prevented** the rest of the code from connecting to the MySQL database. Essentially:

> A **Redis issue** caused a **database connection failure**. Ouch.

---

## Diagnosis

Let’s break down the original logic:

```php
try {
    $redisRateLimiter = new RateLimiter();
    $redisRateLimiter->connect();
    $ip = $_SERVER['REMOTE_ADDR'];
    $conn = null;

    if ($redisRateLimiter->newConnection($ip)) {
        $conn = DatabaseConnection::getInstance()->getConnection();
    } else {
        http_response_code(429);
        echo "Rate limit exceeded. Try again later.";
    }
} catch (Exception $e) {
    error_log("Critical error: " . $e->getMessage());
    echo "A critical error occurred. Please try again later.";
}
```
If Redis failed to connect, newConnection($ip) wouldn’t run — leading to $conn staying null. Then, any function that
relied on $conn (like $conn->prepare(...)) crashed with a fatal error.


The solution was to create a temporary fix by separating the concerns. Redis is optional — a failed connection shouldn't block database access. So I rewrote the code with a fallback:

```php
    $ip = $_SERVER['REMOTE_ADDR'];
    $conn = null;
    $allowConnection = true; // Default to allow
    
    try {
        // Attempt to use Redis for rate limiting
        $redisRateLimiter = new RateLimiter();
        $redisRateLimiter->connect();
    
        if (!$redisRateLimiter->newConnection($ip)) {
            $allowConnection = false;
            http_response_code(429); // Too Many Requests
            echo "Rate limit exceeded. Try again later.";
        }
    } catch (Exception $e) {
        error_log("Redis error: " . $e->getMessage());
        // Continue to DB connection even if Redis fails
    }
    
    // Proceed to DB connection if allowed
    if ($allowConnection) {
        try {
            $dbConnection = new DatabaseConnection();
            $conn = $dbConnection->getConnection();
        } catch (Exception $e) {
            error_log("Database error: " . $e->getMessage());
            echo "[DATABASE] A critical error occurred. Please try again later.";
        }
    }
```
Now:

    Redis failures are logged, but the app stays up.

    Database access proceeds unless Redis explicitly blocks it.

This ensures the function fails gracefully if Redis fails or the DB is unreachable — no more fatal crashes.
Final Cleanup: SQL Data Consistency

And the last issue that I ran into was a change I did a few weeks ago but didn't end up finishing it causing non-equal ID strcutures 
to be stored on the database.

To fix this I changed the command structure for the poop commands and 
instead of using the channel ID provided by StreamElements I switched to the Twitch channel ID. 
This was made possible by changning the property value we're accessing through the `channel` object from
`channel.id` (StreamElements channel ID)
`channel.provider_id` (Twitch channel ID)

```js
${customapi.https://warnstrom.com/API/save_poop.php
?pooper=${sender}
&poop_victim=${random.chatter}
&streamer_id=${channel.provider_id}
&api_key=} 
```

To align with proper Twitch user IDs, I updated legacy UUIDs which were provided by StreamElements in my database:
```sql
UPDATE poop_records
SET streamer_id = 'new_twitch_ID'
WHERE streamer_id = 'old_streamelements_ID';
```

Lessons Learned

Here are the takeaways from this journey:

    Fail gracefully. Optional services like Redis should never crash your core logic.

    Use isolated try/catch blocks for fault tolerance.

    Always check your DB connection before using it.

    Log intelligently. Logs helped pinpoint and solve cascading failures.

    Clean your data model regularly. Fix mismatched or legacy identifiers.

Final Thoughts

What started as a minor feature — a rate limiter — ended up revealing architectural weaknesses. With some thoughtful error handling and restructuring, I was able to fix the problem in a fairly quick manner.
