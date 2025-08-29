# How CSRF Tokens Provide Security

## 🎫 The Concert Ticket Analogy

Imagine a concert where:
- **Session Cookie** = Your general admission wristband (proves you paid)
- **CSRF Token** = Special ticket stub with unique number for each song request
- **Malicious Site** = Someone outside trying to request songs for you

The outsider has no way to know your special ticket number, so they can't make requests on your behalf!

## How CSRF Tokens Work

### 1. **Token Generation**
```php
// Server generates unique token for each user session
$csrfToken = bin2hex(random_bytes(32)); // 64-character random string
$_SESSION['csrf_token'] = $csrfToken;
```

### 2. **Token Embedding**
```html
<!-- Server includes token in every form -->
<form action="/transfer" method="POST">
    <input type="hidden" name="_token" value="a1b2c3d4e5f6...">
    <input name="amount" value="100">
    <button type="submit">Transfer</button>
</form>
```

### 3. **Token Validation**
```php
// Server validates token on every POST request
if ($_POST['_token'] !== $_SESSION['csrf_token']) {
    die('CSRF token mismatch - Request blocked!');
}
```

## Step-by-Step CSRF Protection Flow

### **Step 1: User Visits Protected Page**
```php
// generate_token.php
session_start();

// Generate fresh token if none exists
if (!isset($_SESSION['csrf_token'])) {
    $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
}

$csrfToken = $_SESSION['csrf_token'];
?>

<form method="POST" action="process.php">
    <input type="hidden" name="_token" value="<?= $csrfToken ?>">
    <input name="message" placeholder="Your message">
    <button type="submit">Submit</button>
</form>
```

### **Step 2: User Submits Form**
```
POST /process.php HTTP/1.1
Host: yoursite.com
Cookie: PHPSESSID=abc123
Content-Type: application/x-www-form-urlencoded

_token=a1b2c3d4e5f6789...&message=Hello+World
```

### **Step 3: Server Validates Token**
```php
// process.php
session_start();

function validateCsrfToken() {
    $submittedToken = $_POST['_token'] ?? '';
    $sessionToken = $_SESSION['csrf_token'] ?? '';
    
    // Use hash_equals to prevent timing attacks
    return hash_equals($sessionToken, $submittedToken);
}

if (!validateCsrfToken()) {
    http_response_code(419); // Laravel's CSRF error code
    die('CSRF token mismatch');
}

// Process the legitimate request
echo "Message saved: " . htmlspecialchars($_POST['message']);
```

## Why CSRF Tokens Stop Attacks

### **Attack Scenario Without CSRF Token**
```html
<!-- Malicious site can easily forge requests -->
<form action="https://yoursite.com/transfer" method="POST">
    <input name="amount" value="1000">
    <input name="to_account" value="attacker">
</form>
<script>document.forms[0].submit();</script>
```
**Result:** ✅ Attack succeeds (no token required)

### **Attack Scenario With CSRF Token**
```html
<!-- Malicious site tries to forge request -->
<form action="https://yoursite.com/transfer" method="POST">
    <input name="_token" value="UNKNOWN_TOKEN">
    <input name="amount" value="1000">
    <input name="to_account" value="attacker">
</form>
```
**Result:** ❌ Attack fails (invalid token)

## CSRF Token Implementation Patterns

### **Pattern 1: Per-Session Token**
```php
// One token per user session (simpler)
class SessionCsrfToken {
    public static function generate() {
        if (!isset($_SESSION['csrf_token'])) {
            $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
        }
        return $_SESSION['csrf_token'];
    }
    
    public static function validate($token) {
        return hash_equals($_SESSION['csrf_token'] ?? '', $token);
    }
}
```

### **Pattern 2: Per-Request Token**
```php
// New token for each request (more secure)
class RequestCsrfToken {
    public static function generate() {
        $token = bin2hex(random_bytes(32));
        $_SESSION['csrf_tokens'][] = $token;
        
        // Keep only last 10 tokens
        $_SESSION['csrf_tokens'] = array_slice($_SESSION['csrf_tokens'], -10);
        
        return $token;
    }
    
    public static function validate($token) {
        $tokens = $_SESSION['csrf_tokens'] ?? [];
        
        if (in_array($token, $tokens, true)) {
            // Remove used token
            $_SESSION['csrf_tokens'] = array_diff($tokens, [$token]);
            return true;
        }
        
        return false;
    }
}
```

### **Pattern 3: Time-Based Token**
```php
// Token expires after certain time
class TimedCsrfToken {
    public static function generate($lifetime = 3600) { // 1 hour
        $data = [
            'token' => bin2hex(random_bytes(16)),
            'expires' => time() + $lifetime
        ];
        
        return base64_encode(json_encode($data));
    }
    
    public static function validate($token) {
        $data = json_decode(base64_decode($token), true);
        
        if (!$data || $data['expires'] < time()) {
            return false; // Expired
        }
        
        // Validate against session
        return isset($_SESSION['csrf_base']) && 
               hash_equals($_SESSION['csrf_base'], $data['token']);
    }
}
```

## CSRF Token in Different Contexts

### **HTML Forms**
```html
<form method="POST">
    <input type="hidden" name="_token" value="<?= csrf_token() ?>">
    <!-- form fields -->
</form>
```

### **AJAX Requests**
```javascript
// Include token in AJAX headers
$.ajaxSetup({
    headers: {
        'X-CSRF-TOKEN': $('meta[name="csrf-token"]').attr('content')
    }
});

// Or in request body
fetch('/api/data', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').content
    },
    body: JSON.stringify({data: 'value'})
});
```

### **Meta Tag for JavaScript**
```html
<head>
    <meta name="csrf-token" content="<?= csrf_token() ?>">
</head>
```

## Complete CSRF Protection Example

```php
<?php
// csrf_protection.php - Complete implementation

session_start();

class CsrfProtection {
    private static $tokenName = '_token';
    
    public static function generateToken() {
        if (!isset($_SESSION['csrf_token'])) {
            $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
        }
        return $_SESSION['csrf_token'];
    }
    
    public static function validateToken($token = null) {
        $token = $token ?? ($_POST[self::$tokenName] ?? $_SERVER['HTTP_X_CSRF_TOKEN'] ?? '');
        $sessionToken = $_SESSION['csrf_token'] ?? '';
        
        return hash_equals($sessionToken, $token);
    }
    
    public static function tokenField() {
        return '<input type="hidden" name="' . self::$tokenName . '" value="' . self::generateToken() . '">';
    }
    
    public static function requireValidToken() {
        if ($_SERVER['REQUEST_METHOD'] === 'POST' && !self::validateToken()) {
            http_response_code(419);
            die('CSRF token mismatch');
        }
    }
}

// Auto-protect all POST requests
CsrfProtection::requireValidToken();

// Helper functions
function csrf_token() {
    return CsrfProtection::generateToken();
}

function csrf_field() {
    return CsrfProtection::tokenField();
}
?>

<!-- Usage in HTML -->
<form method="POST" action="process.php">
    <?= csrf_field() ?>
    <input name="data" value="test">
    <button type="submit">Submit</button>
</form>
```

The beauty of CSRF tokens is their simplicity: **if you can't prove you got the token from the legitimate site, you can't make requests to that site!**