# Advanced Session Management

## Custom Session Handlers
```php
<?php
class DatabaseSessionHandler implements SessionHandlerInterface {
    public function open($savePath, $sessionName) {
        // Database connection logic
        return true;
    }
    
    public function read($id) {
        // Read session data from database
        $stmt = $this->pdo->prepare("SELECT data FROM sessions WHERE id = ?");
        $stmt->execute([$id]);
        return $stmt->fetchColumn() ?: '';
    }
    
    public function write($id, $data) {
        // Write session data to database
        $stmt = $this->pdo->prepare("REPLACE INTO sessions (id, data, timestamp) VALUES (?, ?, ?)");
        return $stmt->execute([$id, $data, time()]);
    }
}

session_set_save_handler(new DatabaseSessionHandler());
?>
```

## Session Security Best Practices
```php
<?php
// Secure session configuration
ini_set('session.cookie_lifetime', 0);
ini_set('session.cookie_httponly', 1);
ini_set('session.cookie_secure', 1);
ini_set('session.cookie_samesite', 'Strict');
ini_set('session.use_strict_mode', 1);

// Session timeout handling
function checkSessionTimeout() {
    $timeout = 1800; // 30 minutes
    if (isset($_SESSION['last_activity']) && 
        (time() - $_SESSION['last_activity'] > $timeout)) {
        session_destroy();
        header('Location: login.php');
    }
    $_SESSION['last_activity'] = time();
}
?>
```

## Professional Session Class
```php
<?php
class SessionManager {
    public static function start() {
        if (session_status() === PHP_SESSION_NONE) {
            session_start();
        }
    }
    
    public static function set($key, $value) {
        self::start();
        $_SESSION[$key] = $value;
    }
    
    public static function get($key, $default = null) {
        self::start();
        return $_SESSION[$key] ?? $default;
    }
    
    public static function destroy() {
        self::start();
        session_destroy();
        setcookie(session_name(), '', time() - 3600);
    }
}
?>
```