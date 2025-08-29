# Building Your Own Session System (Laravel-Style)

## 🏗️ The Blueprint: What We're Building
A session system that works like Laravel's but in pure PHP - no framework needed!

## Step 1: Basic Session Manager Class

```php
<?php
// SessionManager.php - Your Laravel-inspired session handler

class SessionManager {
    private $sessionId;
    private $sessionPath;
    private $data = [];
    private $started = false;
    
    public function __construct($sessionPath = './sessions') {
        $this->sessionPath = $sessionPath;
        
        // Create sessions directory if it doesn't exist
        if (!is_dir($this->sessionPath)) {
            mkdir($this->sessionPath, 0755, true);
        }
    }
    
    /**
     * Start the session (like Laravel's StartSession middleware)
     */
    public function start() {
        if ($this->started) {
            return;
        }
        
        // Get session ID from cookie or create new one
        $this->sessionId = $_COOKIE['app_session'] ?? $this->generateSessionId();
        
        // Set the session cookie
        setcookie('app_session', $this->sessionId, [
            'expires' => time() + (120 * 60), // 2 hours like Laravel default
            'path' => '/',
            'httponly' => true,
            'secure' => isset($_SERVER['HTTPS']),
            'samesite' => 'Lax'
        ]);
        
        // Load existing session data
        $this->loadSessionData();
        $this->started = true;
    }
    
    /**
     * Generate unique session ID (Laravel uses Str::random(40))
     */
    private function generateSessionId() {
        return 'sess_' . bin2hex(random_bytes(20)); // 40 character string
    }
    
    /**
     * Load session data from file (like Laravel's file driver)
     */
    private function loadSessionData() {
        $sessionFile = $this->getSessionFilePath();
        
        if (file_exists($sessionFile)) {
            $content = file_get_contents($sessionFile);
            $this->data = unserialize($content) ?: [];
        }
    }
    
    /**
     * Get session file path
     */
    private function getSessionFilePath() {
        return $this->sessionPath . '/' . $this->sessionId;
    }
    
    /**
     * Store data in session (Laravel: session(['key' => 'value']))
     */
    public function put($key, $value) {
        $this->ensureStarted();
        
        // Support dot notation like Laravel
        if (strpos($key, '.') !== false) {
            $this->setNestedValue($this->data, $key, $value);
        } else {
            $this->data[$key] = $value;
        }
    }
    
    /**
     * Get data from session (Laravel: session('key'))
     */
    public function get($key, $default = null) {
        $this->ensureStarted();
        
        if (strpos($key, '.') !== false) {
            return $this->getNestedValue($this->data, $key, $default);
        }
        
        return $this->data[$key] ?? $default;
    }
    
    /**
     * Check if session has key (Laravel: session()->has('key'))
     */
    public function has($key) {
        return $this->get($key) !== null;
    }
    
    /**
     * Remove key from session (Laravel: session()->forget('key'))
     */
    public function forget($key) {
        $this->ensureStarted();
        unset($this->data[$key]);
    }
    
    /**
     * Clear all session data (Laravel: session()->flush())
     */
    public function flush() {
        $this->ensureStarted();
        $this->data = [];
    }
    
    /**
     * Save session data to file (Laravel does this automatically)
     */
    public function save() {
        if (!$this->started) {
            return;
        }
        
        $sessionFile = $this->getSessionFilePath();
        file_put_contents($sessionFile, serialize($this->data), LOCK_EX);
    }
    
    /**
     * Regenerate session ID for security (Laravel: session()->regenerate())
     */
    public function regenerate() {
        $this->ensureStarted();
        
        // Delete old session file
        $oldFile = $this->getSessionFilePath();
        if (file_exists($oldFile)) {
            unlink($oldFile);
        }
        
        // Generate new ID and save
        $this->sessionId = $this->generateSessionId();
        $this->save();
        
        // Update cookie
        setcookie('app_session', $this->sessionId, [
            'expires' => time() + (120 * 60),
            'path' => '/',
            'httponly' => true,
            'secure' => isset($_SERVER['HTTPS']),
            'samesite' => 'Lax'
        ]);
    }
    
    // Helper methods
    private function ensureStarted() {
        if (!$this->started) {
            $this->start();
        }
    }
    
    private function setNestedValue(&$array, $key, $value) {
        $keys = explode('.', $key);
        $current = &$array;
        
        foreach ($keys as $k) {
            if (!isset($current[$k]) || !is_array($current[$k])) {
                $current[$k] = [];
            }
            $current = &$current[$k];
        }
        
        $current = $value;
    }
    
    private function getNestedValue($array, $key, $default = null) {
        $keys = explode('.', $key);
        $current = $array;
        
        foreach ($keys as $k) {
            if (!isset($current[$k])) {
                return $default;
            }
            $current = $current[$k];
        }
        
        return $current;
    }
}
?>
```