# Advanced CSRF Protection Strategies

## 🏰 Building a Fortress: Multiple Defense Layers

Since CSRF tokens have limitations, professional applications use **multiple security layers**. Think of it like a medieval castle - you don't rely on just one wall, you have multiple rings of defense.

## Defense Layer 1: SameSite Cookies

### **How SameSite Works**
```php
// Set session cookie with SameSite attribute
setcookie('session_id', $sessionId, [
    'samesite' => 'Strict',  // Never sent with cross-site requests
    'httponly' => true,      // No JavaScript access
    'secure' => true,        // HTTPS only
    'path' => '/'
]);
```

### **SameSite Options Explained**

```php
// Strict: Maximum protection
'samesite' => 'Strict'
// Cookie NEVER sent with cross-site requests
// Breaks some legitimate use cases (external links)

// Lax: Balanced protection (recommended)
'samesite' => 'Lax'
// Cookie sent with top-level navigation (clicking links)
// NOT sent with embedded requests (images, forms, AJAX)

// None: No protection
'samesite' => 'None'
// Cookie always sent (requires Secure flag)
// Only use when necessary for legitimate cross-site functionality
```

### **SameSite Implementation**
```php
class SecureCookieManager {
    public static function setSessionCookie($name, $value, $options = []) {
        $defaults = [
            'expires' => time() + 3600,
            'path' => '/',
            'domain' => '',
            'secure' => isset($_SERVER['HTTPS']),
            'httponly' => true,
            'samesite' => 'Lax'
        ];
        
        $options = array_merge($defaults, $options);
        setcookie($name, $value, $options);
    }
    
    public static function setStrictCookie($name, $value) {
        self::setSessionCookie($name, $value, ['samesite' => 'Strict']);
    }
}

// Usage
SecureCookieManager::setStrictCookie('session_id', $sessionId);
```

## Defense Layer 2: Origin/Referer Validation

### **Origin Header Validation**
```php
class OriginValidator {
    private static $allowedOrigins = [
        'https://yoursite.com',
        'https://www.yoursite.com',
        'https://app.yoursite.com'
    ];
    
    public static function validateOrigin() {
        $origin = $_SERVER['HTTP_ORIGIN'] ?? '';
        $referer = $_SERVER['HTTP_REFERER'] ?? '';
        
        // Check Origin header first (more reliable)
        if ($origin) {
            return in_array($origin, self::$allowedOrigins, true);
        }
        
        // Fallback to Referer header
        if ($referer) {
            $refererHost = parse_url($referer, PHP_URL_HOST);
            $allowedHosts = array_map(function($url) {
                return parse_url($url, PHP_URL_HOST);
            }, self::$allowedOrigins);
            
            return in_array($refererHost, $allowedHosts, true);
        }
        
        // No Origin or Referer - suspicious
        return false;
    }
    
    public static function requireValidOrigin() {
        if ($_SERVER['REQUEST_METHOD'] === 'POST' && !self::validateOrigin()) {
            http_response_code(403);
            die('Invalid origin');
        }
    }
}
```

### **Smart Origin Checking**
```php
class SmartOriginChecker {
    public static function isValidRequest() {
        // Allow same-origin requests
        if (self::isSameOrigin()) {
            return true;
        }
        
        // For cross-origin, require CSRF token
        if (self::isCrossOrigin()) {
            return CsrfProtection::validateToken();
        }
        
        // No origin info - require CSRF token
        return CsrfProtection::validateToken();
    }
    
    private static function isSameOrigin() {
        $origin = $_SERVER['HTTP_ORIGIN'] ?? '';
        $host = $_SERVER['HTTP_HOST'] ?? '';
        $scheme = isset($_SERVER['HTTPS']) ? 'https' : 'http';
        
        return $origin === "{$scheme}://{$host}";
    }
    
    private static function isCrossOrigin() {
        return isset($_SERVER['HTTP_ORIGIN']) && !self::isSameOrigin();
    }
}
```

## Defense Layer 3: Double Submit Cookie Pattern

### **How Double Submit Works**
```php
class DoubleSubmitCsrf {
    private static $cookieName = 'csrf_token';
    
    public static function generateToken() {
        $token = bin2hex(random_bytes(32));
        
        // Store in both session AND cookie
        $_SESSION['csrf_token'] = $token;
        setcookie(self::$cookieName, $token, [
            'httponly' => false, // JavaScript needs access
            'secure' => isset($_SERVER['HTTPS']),
            'samesite' => 'Strict'
        ]);
        
        return $token;
    }
    
    public static function validateToken() {
        $sessionToken = $_SESSION['csrf_token'] ?? '';
        $cookieToken = $_COOKIE[self::$cookieName] ?? '';
        $submittedToken = $_POST['_token'] ?? $_SERVER['HTTP_X_CSRF_TOKEN'] ?? '';
        
        // All three must match
        return hash_equals($sessionToken, $submittedToken) && 
               hash_equals($sessionToken, $cookieToken);
    }
}
```

### **JavaScript Integration**
```javascript
// Automatically include CSRF token in AJAX requests
function getCsrfToken() {
    return document.cookie
        .split('; ')
        .find(row => row.startsWith('csrf_token='))
        ?.split('=')[1];
}

// Set up AJAX defaults
$.ajaxSetup({
    beforeSend: function(xhr) {
        xhr.setRequestHeader('X-CSRF-TOKEN', getCsrfToken());
    }
});

// Or with fetch
function securePost(url, data) {
    return fetch(url, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-CSRF-TOKEN': getCsrfToken()
        },
        body: JSON.stringify(data)
    });
}
```

## Defense Layer 4: Custom Headers

### **Requiring Custom Headers**
```php
class CustomHeaderProtection {
    private static $requiredHeader = 'X-Requested-With';
    private static $expectedValue = 'XMLHttpRequest';
    
    public static function validateCustomHeader() {
        $headerValue = $_SERVER['HTTP_X_REQUESTED_WITH'] ?? '';
        return $headerValue === self::$expectedValue;
    }
    
    public static function requireCustomHeader() {
        if ($_SERVER['REQUEST_METHOD'] === 'POST' && 
            !self::validateCustomHeader()) {
            http_response_code(400);
            die('Missing required header');
        }
    }
}
```

### **JavaScript Implementation**
```javascript
// All AJAX requests include custom header
function makeSecureRequest(url, data) {
    return fetch(url, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-Requested-With': 'XMLHttpRequest', // Custom header
            'X-CSRF-TOKEN': getCsrfToken()        // CSRF token
        },
        body: JSON.stringify(data)
    });
}
```

## Defense Layer 5: Content Security Policy (CSP)

### **CSP Headers for CSRF Protection**
```php
class CspProtection {
    public static function setHeaders() {
        // Prevent inline scripts (stops XSS that could steal tokens)
        header("Content-Security-Policy: " . implode('; ', [
            "default-src 'self'",
            "script-src 'self' 'unsafe-eval'", // No 'unsafe-inline'
            "style-src 'self' 'unsafe-inline'",
            "img-src 'self' data: https:",
            "connect-src 'self'",
            "font-src 'self'",
            "object-src 'none'",
            "base-uri 'self'",
            "form-action 'self'" // Only allow forms to submit to same origin
        ]));
    }
}

// Apply CSP to all pages
CspProtection::setHeaders();
```

## Defense Layer 6: Rate Limiting

### **Request Rate Limiting**
```php
class RateLimiter {
    private static $redis;
    
    public static function init() {
        self::$redis = new Redis();
        self::$redis->connect('127.0.0.1', 6379);
    }
    
    public static function checkLimit($identifier, $maxRequests = 10, $timeWindow = 60) {
        $key = "rate_limit:{$identifier}";
        $current = self::$redis->incr($key);
        
        if ($current === 1) {
            self::$redis->expire($key, $timeWindow);
        }
        
        return $current <= $maxRequests;
    }
    
    public static function limitCsrfAttempts() {
        $ip = $_SERVER['REMOTE_ADDR'];
        $userId = $_SESSION['user_id'] ?? 'anonymous';
        $identifier = "{$ip}:{$userId}";
        
        if (!self::checkLimit($identifier, 5, 300)) { // 5 attempts per 5 minutes
            http_response_code(429);
            die('Too many requests');
        }
    }
}
```

## Complete Multi-Layer Protection

### **Comprehensive CSRF Protection Class**
```php
class AdvancedCsrfProtection {
    public static function protect() {
        // Layer 1: SameSite cookies (set during login)
        // Layer 2: Origin validation
        OriginValidator::requireValidOrigin();
        
        // Layer 3: Rate limiting
        RateLimiter::limitCsrfAttempts();
        
        // Layer 4: Custom header for AJAX
        if (self::isAjaxRequest()) {
            CustomHeaderProtection::requireCustomHeader();
        }
        
        // Layer 5: CSRF token validation
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            if (!DoubleSubmitCsrf::validateToken()) {
                self::handleCsrfFailure();
            }
        }
        
        // Layer 6: CSP headers
        CspProtection::setHeaders();
    }
    
    private static function isAjaxRequest() {
        return !empty($_SERVER['HTTP_X_REQUESTED_WITH']) ||
               strpos($_SERVER['CONTENT_TYPE'] ?? '', 'application/json') !== false;
    }
    
    private static function handleCsrfFailure() {
        // Log the attempt
        error_log("CSRF attack attempt from " . $_SERVER['REMOTE_ADDR']);
        
        // Rate limit the IP
        RateLimiter::limitCsrfAttempts();
        
        // Return error
        http_response_code(419);
        die('CSRF token mismatch');
    }
}

// Apply protection to all requests
AdvancedCsrfProtection::protect();
```

## Mobile App Considerations

### **API Token-Based Protection**
```php
class ApiCsrfProtection {
    public static function generateApiToken($userId) {
        $payload = [
            'user_id' => $userId,
            'timestamp' => time(),
            'nonce' => bin2hex(random_bytes(16))
        ];
        
        $signature = hash_hmac('sha256', json_encode($payload), API_SECRET);
        return base64_encode(json_encode($payload)) . '.' . $signature;
    }
    
    public static function validateApiToken($token) {
        $parts = explode('.', $token);
        if (count($parts) !== 2) return false;
        
        $payload = json_decode(base64_decode($parts[0]), true);
        $signature = $parts[1];
        
        // Verify signature
        $expectedSignature = hash_hmac('sha256', json_encode($payload), API_SECRET);
        if (!hash_equals($expectedSignature, $signature)) return false;
        
        // Check timestamp (token expires in 1 hour)
        if (time() - $payload['timestamp'] > 3600) return false;
        
        return $payload;
    }
}
```

## Testing Your CSRF Protection

### **CSRF Test Suite**
```php
class CsrfTestSuite {
    public static function runTests() {
        echo "Testing CSRF Protection...\n";
        
        // Test 1: Valid token
        self::testValidToken();
        
        // Test 2: Invalid token
        self::testInvalidToken();
        
        // Test 3: Missing token
        self::testMissingToken();
        
        // Test 4: Cross-origin request
        self::testCrossOrigin();
        
        echo "All tests completed.\n";
    }
    
    private static function testValidToken() {
        // Simulate valid request with proper token
        $_POST['_token'] = $_SESSION['csrf_token'];
        $result = CsrfProtection::validateToken();
        assert($result === true, "Valid token test failed");
        echo "✓ Valid token test passed\n";
    }
    
    // Additional test methods...
}
```

By implementing multiple layers of protection, you create a robust defense system that can withstand various attack vectors, even if one layer fails.