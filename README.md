# From Errors to Elegance: Handling Redis Rate Limiting and Database Connections in PHP

Working on high-availability PHP applications always presents one truth: every dependency you introduce can fail. 

---

## The Initial Setup

My application collects and serves data using a MySQL database. To avoid abuse and excessive traffic, I implemented a rate limiter using Redis.

In PHP, I was using a custom `RateLimiter` class that connected to Redis and tracked IP requests. Everything worked beautifully — **until it didn’t**.

I initially overlooked the fact that if the RateLimiter service goes down or fails to start correctly, it would prevent any connection to the database. 
While this might be acceptable in some scenarios, it's not the desired behavior for this particular use case.

---

## The Problem

Once in production, I started getting a **critical error** message:
```
A critical error occurred. Please try again later.
Fatal error: Uncaught Error: Call to a member function prepare() on null
``` 

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

## 🛠️ Attempted Refactor: DatabaseConnection Singleton

While refining the error handling logic, I tried optimizing the `DatabaseConnection` class to follow the Singleton pattern. The idea was to create a single persistent database connection instance that could be reused throughout the application.

### 💡 The Updated Class Concept

```php
class DatabaseConnection {
    private static $instance = null;
    private $conn;
    private $servername = "";
    private $username = "";
    private $password = "";
    private $dbname = "";

    private function __construct() {
        $this->conn = new mysqli(hostname: $this->servername, username: $this->username, password: $this->password, database: $this->dbname);
        if ($this->conn->connect_error) {
            throw new Exception("Database connection failed: " . $this->conn->connect_error);
        }
    }

    public static function getInstance() {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    public function getConnection() {
        return $this->conn;
    }
}
```
What Went Wrong

Despite implementing this cleaner structure, it didn’t work as expected in the live environment.
The Issues:

    Silent failure or null connection if Redis failed or the class was instantiated incorrectly.

    Lack of error propagation: The connection failure wasn't bubbling up properly to the main logic.

Ultimately, the class added more confusion than value in this specific use case.
Final Decision
I reverted from using the newer instantiation method:

```php
$conn = DatabaseConnection::getInstance()->getConnection();
```

back to the simpler, more straightforward approach:

```php
$dbConnection = new DatabaseConnection();
$conn = $dbConnection->getConnection();
```

This change helped avoid some unexpected issues tied to the Singleton implementation.

The final issue I encountered was related to a change I had made weeks ago but never fully implemented. This resulted in inconsistent ID formats being stored in the database.

To resolve this, I updated the command structure used by the poop commands. Previously, it relied on the StreamElements channel ID (channel.id). I’ve now switched it to use the Twitch channel ID (channel.provider_id) instead, ensuring all IDs follow a consistent structure.

Here’s the updated command:
```js
${customapi.https://warnstrom.com/API/save_poop.php
?pooper=${sender}
&poop_victim=${random.chatter}
&streamer_id=${channel.provider_id}
&api_key=}
```

This change not only standardizes the stored IDs but also aligns better with Twitch's platform-specific identifiers, reducing the risk of mismatches across services.

To align with proper Twitch user IDs, I updated legacy UUIDs which were provided by StreamElements in my database:
```sql
UPDATE poop_records
SET streamer_id = 'new_twitch_ID'
WHERE streamer_id = 'old_streamelements_ID';
```

Lessons Learned

Here are the takeaways from this journey:

    Fail gracefully. Optional services like Redis should never crash your core logic.
    
    Don't rewrite code and deploy it to production without testing it thoroughly before hand.

    Log intelligently. Logs helped pinpoint and solve cascading failures.

    Clean your data model regularly. Fix mismatched or legacy identifiers.

Final Thoughts

What started as a minor feature — a rate limiter — ended up revealing architectural weaknesses. With some thoughtful error handling and restructuring, I was able to fix the problem in a fairly quick manner.
