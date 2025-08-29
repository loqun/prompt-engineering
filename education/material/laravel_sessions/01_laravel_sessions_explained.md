# Laravel Sessions - How They Really Work

## 🍪 The Cookie Jar Analogy
Imagine Laravel sessions like a **smart cookie jar system**:
- Your browser gets a **special ticket number** (session ID)
- Laravel keeps a **filing cabinet** (session storage) with folders labeled by ticket numbers
- When you visit again, you show your ticket, Laravel finds your folder with all your stuff

## How Laravel Sessions Work Under the Hood

### 1. Session Drivers (Storage Methods)
```php
// config/session.php
'driver' => 'file', // Options: file, cookie, database, memcached, redis, array
```

**File Driver** (Default):
- Sessions stored in `storage/framework/sessions/`
- Each session = one file named by session ID
- File contains serialized PHP data

**Database Driver**:
- Sessions stored in database table
- Better for multiple servers
- Easier to manage and query

### 2. Session Lifecycle in Laravel

```php
// 1. Middleware starts session
StartSession::class

// 2. Session ID created/retrieved
$sessionId = session()->getId(); // sess_abc123def456

// 3. Data stored/retrieved
session(['user_id' => 123]);
$userId = session('user_id');

// 4. Session saved automatically at request end
```

## Laravel Session Configuration Deep Dive

```php
// config/session.php - Key settings explained
return [
    'driver' => 'file',                    // Where sessions are stored
    'lifetime' => 120,                     // Minutes before expiry
    'expire_on_close' => false,            // Expire when browser closes?
    'encrypt' => false,                    // Encrypt session data?
    'files' => storage_path('framework/sessions'), // File storage location
    'connection' => null,                  // Database connection for DB driver
    'table' => 'sessions',                 // Database table name
    'store' => null,                       // Cache store for cache driver
    'lottery' => [2, 100],                 // Garbage collection odds
    'cookie' => 'laravel_session',         // Session cookie name
    'path' => '/',                         // Cookie path
    'domain' => null,                      // Cookie domain
    'secure' => false,                     // HTTPS only?
    'http_only' => true,                   // JavaScript access blocked?
    'same_site' => 'lax',                  // CSRF protection level
];
```

## Laravel Session Methods You Need to Know

```php
// Store data
session(['key' => 'value']);
session()->put('user.name', 'John');

// Retrieve data
$value = session('key');
$name = session('user.name', 'Guest'); // With default

// Check existence
if (session()->has('user_id')) {
    // User is logged in
}

// Remove data
session()->forget('key');
session()->flush(); // Clear all

// Flash data (available only next request)
session()->flash('message', 'Success!');

// Regenerate ID (security)
session()->regenerate();
```