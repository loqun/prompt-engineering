# Laravel-Style Helper Functions for Your Barebone App

## 🛠️ Creating Laravel's session() Helper

```php
<?php
// session_helpers.php - Global helper functions like Laravel

// Global session manager instance
$globalSessionManager = null;

/**
 * Get the session manager instance
 */
function getSessionManager() {
    global $globalSessionManager;
    
    if ($globalSessionManager === null) {
        $globalSessionManager = new SessionManager();
        $globalSessionManager->start();
    }
    
    return $globalSessionManager;
}

/**
 * Laravel-style session() helper function
 * 
 * Usage examples:
 * session(['user_id' => 123])           // Store data
 * session('user_id')                    // Get data
 * session('user_id', 'guest')           // Get with default
 * session()->flush()                    // Access manager directly
 */
function session($key = null, $default = null) {
    $manager = getSessionManager();
    
    // If no arguments, return the manager instance
    if ($key === null) {
        return $manager;
    }
    
    // If key is array, store multiple values
    if (is_array($key)) {
        foreach ($key as $k => $v) {
            $manager->put($k, $v);
        }
        return;
    }
    
    // If only key provided, get the value
    if ($default === null && func_num_args() === 1) {
        return $manager->get($key);
    }
    
    // Get with default value
    return $manager->get($key, $default);
}

/**
 * Flash data helper (available only next request)
 */
function flash($key, $value = null) {
    if ($value === null) {
        // Get flash data
        $flashData = session('_flash.' . $key);
        session()->forget('_flash.' . $key);
        return $flashData;
    }
    
    // Set flash data
    session(['_flash.' . $key => $value]);
}

/**
 * Old input helper (for form repopulation)
 */
function old($key, $default = null) {
    return flash('old_input.' . $key) ?? $default;
}

/**
 * Store old input data
 */
function flashInput($input) {
    foreach ($input as $key => $value) {
        flash('old_input.' . $key, $value);
    }
}

/**
 * CSRF token generation and validation
 */
function csrf_token() {
    if (!session()->has('_token')) {
        session(['_token' => bin2hex(random_bytes(20))]);
    }
    return session('_token');
}

function csrf_field() {
    return '<input type="hidden" name="_token" value="' . csrf_token() . '">';
}

function verify_csrf($token) {
    return hash_equals(session('_token', ''), $token);
}

// Auto-save session at script end
register_shutdown_function(function() {
    global $globalSessionManager;
    if ($globalSessionManager !== null) {
        $globalSessionManager->save();
    }
});
?>
```

## 🚀 Using Your Session System

### Basic Usage Example
```php
<?php
require_once 'SessionManager.php';
require_once 'session_helpers.php';

// Store user login
session(['user_id' => 123, 'username' => 'john_doe']);

// Get user data
$userId = session('user_id');
$username = session('username', 'Guest');

// Check if logged in
if (session()->has('user_id')) {
    echo "Welcome back, " . session('username');
} else {
    echo "Please log in";
}

// Nested data (Laravel-style dot notation)
session(['user.profile.name' => 'John Doe']);
$name = session('user.profile.name');
?>
```

### Login System Example
```php
<?php
// login.php
require_once 'session_helpers.php';

if ($_POST['login'] ?? false) {
    $username = $_POST['username'];
    $password = $_POST['password'];
    
    // Validate credentials (your logic here)
    if (authenticate($username, $password)) {
        // Regenerate session ID for security
        session()->regenerate();
        
        // Store user data
        session([
            'user_id' => $user['id'],
            'username' => $user['username'],
            'logged_in' => true
        ]);
        
        flash('success', 'Login successful!');
        header('Location: dashboard.php');
        exit;
    } else {
        flash('error', 'Invalid credentials');
        flashInput($_POST); // For form repopulation
    }
}
?>

<!DOCTYPE html>
<html>
<head>
    <title>Login</title>
</head>
<body>
    <?php if (flash('error')): ?>
        <div class="error"><?= flash('error') ?></div>
    <?php endif; ?>
    
    <form method="POST">
        <?= csrf_field() ?>
        <input type="text" name="username" value="<?= old('username') ?>" placeholder="Username">
        <input type="password" name="password" placeholder="Password">
        <button type="submit" name="login">Login</button>
    </form>
</body>
</html>
```

### Dashboard with Session Check
```php
<?php
// dashboard.php
require_once 'session_helpers.php';

// Check if user is logged in
if (!session('logged_in')) {
    header('Location: login.php');
    exit;
}

$username = session('username');
?>

<!DOCTYPE html>
<html>
<head>
    <title>Dashboard</title>
</head>
<body>
    <h1>Welcome, <?= htmlspecialchars($username) ?>!</h1>
    
    <?php if (flash('success')): ?>
        <div class="success"><?= flash('success') ?></div>
    <?php endif; ?>
    
    <a href="logout.php">Logout</a>
</body>
</html>
```

### Logout System
```php
<?php
// logout.php
require_once 'session_helpers.php';

// Clear all session data
session()->flush();

// Regenerate session ID
session()->regenerate();

flash('info', 'You have been logged out');
header('Location: login.php');
?>
```