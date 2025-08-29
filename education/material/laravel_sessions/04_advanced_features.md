# Advanced Session Features (Laravel-Level)

## 🔒 Database Session Driver

```php
<?php
// DatabaseSessionManager.php - Store sessions in database like Laravel

class DatabaseSessionManager extends SessionManager {
    private $pdo;
    private $table = 'sessions';
    
    public function __construct($pdo, $table = 'sessions') {
        $this->pdo = $pdo;
        $this->table = $table;
        $this->createSessionsTable();
    }
    
    /**
     * Create sessions table (Laravel migration equivalent)
     */
    private function createSessionsTable() {
        $sql = "CREATE TABLE IF NOT EXISTS {$this->table} (
            id VARCHAR(255) PRIMARY KEY,
            user_id INT NULL,
            ip_address VARCHAR(45) NULL,
            user_agent TEXT NULL,
            payload LONGTEXT NOT NULL,
            last_activity INT NOT NULL,
            INDEX sessions_user_id_index (user_id),
            INDEX sessions_last_activity_index (last_activity)
        )";
        
        $this->pdo->exec($sql);
    }
    
    /**
     * Load session data from database
     */
    protected function loadSessionData() {
        $stmt = $this->pdo->prepare("SELECT payload FROM {$this->table} WHERE id = ? AND last_activity > ?");
        $stmt->execute([$this->sessionId, time() - (120 * 60)]); // 2 hour expiry
        
        $result = $stmt->fetch(PDO::FETCH_ASSOC);
        if ($result) {
            $this->data = unserialize(base64_decode($result['payload'])) ?: [];
        }
    }
    
    /**
     * Save session data to database
     */
    public function save() {
        if (!$this->started) {
            return;
        }
        
        $payload = base64_encode(serialize($this->data));
        $userId = $this->data['user_id'] ?? null;
        $ipAddress = $_SERVER['REMOTE_ADDR'] ?? null;
        $userAgent = $_SERVER['HTTP_USER_AGENT'] ?? null;
        
        $sql = "INSERT INTO {$this->table} (id, user_id, ip_address, user_agent, payload, last_activity) 
                VALUES (?, ?, ?, ?, ?, ?) 
                ON DUPLICATE KEY UPDATE 
                user_id = VALUES(user_id),
                ip_address = VALUES(ip_address),
                user_agent = VALUES(user_agent),
                payload = VALUES(payload),
                last_activity = VALUES(last_activity)";
        
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute([
            $this->sessionId,
            $userId,
            $ipAddress,
            $userAgent,
            $payload,
            time()
        ]);
    }
    
    /**
     * Clean up expired sessions (Laravel's garbage collection)
     */
    public function garbageCollection() {
        $expiredTime = time() - (120 * 60); // 2 hours ago
        $stmt = $this->pdo->prepare("DELETE FROM {$this->table} WHERE last_activity < ?");
        $stmt->execute([$expiredTime]);
    }
}
?>
```

## 🛡️ Session Security Middleware

```php
<?php
// SessionSecurityMiddleware.php - Laravel-style middleware

class SessionSecurityMiddleware {
    private $sessionManager;
    
    public function __construct($sessionManager) {
        $this->sessionManager = $sessionManager;
    }
    
    /**
     * Handle request (like Laravel middleware)
     */
    public function handle($request, $next) {
        // Start session
        $this->sessionManager->start();
        
        // CSRF protection for POST requests
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            $this->verifyCsrfToken();
        }
        
        // Session hijacking protection
        $this->validateSessionFingerprint();
        
        // Session timeout check
        $this->checkSessionTimeout();
        
        // Continue to next middleware/controller
        $response = $next($request);
        
        // Save session after request
        $this->sessionManager->save();
        
        return $response;
    }
    
    /**
     * Verify CSRF token
     */
    private function verifyCsrfToken() {
        $token = $_POST['_token'] ?? $_SERVER['HTTP_X_CSRF_TOKEN'] ?? null;
        
        if (!verify_csrf($token)) {
            http_response_code(419);
            die('CSRF token mismatch');
        }
    }
    
    /**
     * Validate session fingerprint (prevent session hijacking)
     */
    private function validateSessionFingerprint() {
        $currentFingerprint = $this->generateFingerprint();
        $storedFingerprint = session('_fingerprint');
        
        if ($storedFingerprint === null) {
            // First time, store fingerprint
            session(['_fingerprint' => $currentFingerprint]);
        } elseif ($storedFingerprint !== $currentFingerprint) {
            // Fingerprint mismatch, possible hijacking
            session()->flush();
            session()->regenerate();
            session(['_fingerprint' => $currentFingerprint]);
        }
    }
    
    /**
     * Generate session fingerprint
     */
    private function generateFingerprint() {
        $userAgent = $_SERVER['HTTP_USER_AGENT'] ?? '';
        $acceptLanguage = $_SERVER['HTTP_ACCEPT_LANGUAGE'] ?? '';
        $acceptEncoding = $_SERVER['HTTP_ACCEPT_ENCODING'] ?? '';
        
        return hash('sha256', $userAgent . $acceptLanguage . $acceptEncoding);
    }
    
    /**
     * Check session timeout
     */
    private function checkSessionTimeout() {
        $lastActivity = session('_last_activity');
        $timeout = 30 * 60; // 30 minutes
        
        if ($lastActivity && (time() - $lastActivity > $timeout)) {
            session()->flush();
            session()->regenerate();
            flash('warning', 'Session expired due to inactivity');
        }
        
        session(['_last_activity' => time()]);
    }
}
?>
```

## 🎯 Session-Based Shopping Cart

```php
<?php
// ShoppingCart.php - Laravel-style session cart

class ShoppingCart {
    private $sessionKey = 'shopping_cart';
    
    /**
     * Add item to cart
     */
    public function add($productId, $quantity = 1, $options = []) {
        $cart = $this->getCart();
        $itemId = $this->generateItemId($productId, $options);
        
        if (isset($cart[$itemId])) {
            $cart[$itemId]['quantity'] += $quantity;
        } else {
            $cart[$itemId] = [
                'product_id' => $productId,
                'quantity' => $quantity,
                'options' => $options,
                'added_at' => time()
            ];
        }
        
        session([$this->sessionKey => $cart]);
        return $itemId;
    }
    
    /**
     * Remove item from cart
     */
    public function remove($itemId) {
        $cart = $this->getCart();
        unset($cart[$itemId]);
        session([$this->sessionKey => $cart]);
    }
    
    /**
     * Update item quantity
     */
    public function update($itemId, $quantity) {
        $cart = $this->getCart();
        
        if ($quantity <= 0) {
            $this->remove($itemId);
        } else {
            $cart[$itemId]['quantity'] = $quantity;
            session([$this->sessionKey => $cart]);
        }
    }
    
    /**
     * Get cart contents
     */
    public function getCart() {
        return session($this->sessionKey, []);
    }
    
    /**
     * Get cart count
     */
    public function count() {
        return array_sum(array_column($this->getCart(), 'quantity'));
    }
    
    /**
     * Clear entire cart
     */
    public function clear() {
        session()->forget($this->sessionKey);
    }
    
    /**
     * Generate unique item ID
     */
    private function generateItemId($productId, $options) {
        return md5($productId . serialize($options));
    }
}

// Usage example
$cart = new ShoppingCart();

// Add products
$cart->add(123, 2, ['size' => 'L', 'color' => 'red']);
$cart->add(456, 1);

// Get cart
$cartItems = $cart->getCart();
$totalItems = $cart->count();
?>
```

## 🔄 Session Migration Helper

```php
<?php
// SessionMigrator.php - Migrate from PHP sessions to your system

class SessionMigrator {
    private $sessionManager;
    
    public function __construct($sessionManager) {
        $this->sessionManager = $sessionManager;
    }
    
    /**
     * Migrate from PHP $_SESSION to custom session
     */
    public function migrateFromPhpSession() {
        if (session_status() === PHP_SESSION_ACTIVE) {
            foreach ($_SESSION as $key => $value) {
                $this->sessionManager->put($key, $value);
            }
            
            // Clear PHP session
            session_destroy();
        }
    }
    
    /**
     * Export session data for backup
     */
    public function exportSession($userId) {
        return [
            'user_id' => $userId,
            'session_data' => session()->all(),
            'exported_at' => date('Y-m-d H:i:s'),
            'session_id' => session()->getId()
        ];
    }
    
    /**
     * Import session data from backup
     */
    public function importSession($exportedData) {
        foreach ($exportedData['session_data'] as $key => $value) {
            session([$key => $value]);
        }
    }
}
?>
```