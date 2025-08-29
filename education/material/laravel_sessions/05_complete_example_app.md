# Complete Barebone App with Laravel-Style Sessions

## 🏗️ Project Structure
```
your_app/
├── classes/
│   ├── SessionManager.php
│   ├── DatabaseSessionManager.php
│   └── SessionSecurityMiddleware.php
├── helpers/
│   └── session_helpers.php
├── public/
│   ├── index.php
│   ├── login.php
│   ├── dashboard.php
│   └── logout.php
├── config/
│   └── database.php
└── sessions/ (for file-based sessions)
```

## 📁 Complete Implementation Files

### config/database.php
```php
<?php
return [
    'host' => 'localhost',
    'dbname' => 'your_app',
    'username' => 'root',
    'password' => '',
    'charset' => 'utf8mb4'
];
```

### public/index.php (Bootstrap file)
```php
<?php
// Bootstrap your application
require_once '../classes/SessionManager.php';
require_once '../classes/DatabaseSessionManager.php';
require_once '../classes/SessionSecurityMiddleware.php';
require_once '../helpers/session_helpers.php';

// Database connection
$config = require '../config/database.php';
try {
    $pdo = new PDO(
        "mysql:host={$config['host']};dbname={$config['dbname']};charset={$config['charset']}",
        $config['username'],
        $config['password'],
        [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
    );
} catch (PDOException $e) {
    die('Database connection failed: ' . $e->getMessage());
}

// Initialize session manager (choose file or database)
$useDatabase = true; // Set to false for file-based sessions

if ($useDatabase) {
    $globalSessionManager = new DatabaseSessionManager($pdo);
} else {
    $globalSessionManager = new SessionManager('../sessions');
}

// Initialize security middleware
$middleware = new SessionSecurityMiddleware($globalSessionManager);

// Simple router
$request = $_SERVER['REQUEST_URI'];
$path = parse_url($request, PHP_URL_PATH);

// Handle request through middleware
$middleware->handle($request, function($request) use ($path) {
    switch ($path) {
        case '/':
        case '/index.php':
            include 'home.php';
            break;
        case '/login.php':
            include 'login.php';
            break;
        case '/dashboard.php':
            include 'dashboard.php';
            break;
        case '/logout.php':
            include 'logout.php';
            break;
        default:
            http_response_code(404);
            echo '404 Not Found';
    }
});
?>
```

### public/home.php
```php
<!DOCTYPE html>
<html>
<head>
    <title>Laravel-Style Sessions Demo</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
        .alert { padding: 10px; margin: 10px 0; border-radius: 4px; }
        .success { background: #d4edda; color: #155724; border: 1px solid #c3e6cb; }
        .error { background: #f8d7da; color: #721c24; border: 1px solid #f5c6cb; }
        .info { background: #d1ecf1; color: #0c5460; border: 1px solid #bee5eb; }
    </style>
</head>
<body>
    <h1>Laravel-Style Sessions Demo</h1>
    
    <?php if (flash('info')): ?>
        <div class="alert info"><?= htmlspecialchars(flash('info')) ?></div>
    <?php endif; ?>
    
    <?php if (session('logged_in')): ?>
        <p>Welcome back, <strong><?= htmlspecialchars(session('username')) ?></strong>!</p>
        <p><a href="/dashboard.php">Go to Dashboard</a> | <a href="/logout.php">Logout</a></p>
    <?php else: ?>
        <p>You are not logged in.</p>
        <p><a href="/login.php">Login</a></p>
    <?php endif; ?>
    
    <h2>Session Information</h2>
    <p><strong>Session ID:</strong> <?= session()->getId() ?></p>
    <p><strong>Items in Session:</strong> <?= count(session()->all()) ?></p>
    
    <h3>Current Session Data:</h3>
    <pre><?= htmlspecialchars(print_r(session()->all(), true)) ?></pre>
</body>
</html>
```

### public/login.php
```php
<?php
// Handle login form submission
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['login'])) {
    $username = trim($_POST['username']);
    $password = $_POST['password'];
    
    // Simple authentication (replace with your logic)
    $users = [
        'admin' => 'password123',
        'user' => 'secret456'
    ];
    
    if (isset($users[$username]) && $users[$username] === $password) {
        // Regenerate session ID for security
        session()->regenerate();
        
        // Store user data
        session([
            'user_id' => array_search($username, array_keys($users)) + 1,
            'username' => $username,
            'logged_in' => true,
            'login_time' => time()
        ]);
        
        flash('success', 'Login successful! Welcome back.');
        header('Location: /dashboard.php');
        exit;
    } else {
        flash('error', 'Invalid username or password.');
        flashInput(['username' => $username]); // Remember username
    }
}
?>

<!DOCTYPE html>
<html>
<head>
    <title>Login - Laravel Sessions Demo</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 400px; margin: 100px auto; padding: 20px; }
        .form-group { margin-bottom: 15px; }
        label { display: block; margin-bottom: 5px; font-weight: bold; }
        input[type="text"], input[type="password"] { width: 100%; padding: 8px; border: 1px solid #ddd; border-radius: 4px; }
        button { background: #007bff; color: white; padding: 10px 20px; border: none; border-radius: 4px; cursor: pointer; }
        button:hover { background: #0056b3; }
        .alert { padding: 10px; margin: 10px 0; border-radius: 4px; }
        .error { background: #f8d7da; color: #721c24; border: 1px solid #f5c6cb; }
    </style>
</head>
<body>
    <h2>Login</h2>
    
    <?php if (flash('error')): ?>
        <div class="alert error"><?= htmlspecialchars(flash('error')) ?></div>
    <?php endif; ?>
    
    <form method="POST">
        <?= csrf_field() ?>
        
        <div class="form-group">
            <label for="username">Username:</label>
            <input type="text" id="username" name="username" value="<?= htmlspecialchars(old('username', '')) ?>" required>
        </div>
        
        <div class="form-group">
            <label for="password">Password:</label>
            <input type="password" id="password" name="password" required>
        </div>
        
        <button type="submit" name="login">Login</button>
    </form>
    
    <p><small>Demo users: admin/password123 or user/secret456</small></p>
    <p><a href="/">Back to Home</a></p>
</body>
</html>
```

### public/dashboard.php
```php
<?php
// Check authentication
if (!session('logged_in')) {
    flash('error', 'Please log in to access the dashboard.');
    header('Location: /login.php');
    exit;
}

// Handle cart actions
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    if (isset($_POST['add_to_cart'])) {
        $cart = session('cart', []);
        $productId = $_POST['product_id'];
        $cart[$productId] = ($cart[$productId] ?? 0) + 1;
        session(['cart' => $cart]);
        flash('success', 'Item added to cart!');
    }
    
    if (isset($_POST['clear_cart'])) {
        session()->forget('cart');
        flash('info', 'Cart cleared!');
    }
}

$username = session('username');
$loginTime = session('login_time');
$cart = session('cart', []);
?>

<!DOCTYPE html>
<html>
<head>
    <title>Dashboard - Laravel Sessions Demo</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
        .header { display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #ddd; padding-bottom: 10px; }
        .alert { padding: 10px; margin: 10px 0; border-radius: 4px; }
        .success { background: #d4edda; color: #155724; border: 1px solid #c3e6cb; }
        .info { background: #d1ecf1; color: #0c5460; border: 1px solid #bee5eb; }
        .product { border: 1px solid #ddd; padding: 15px; margin: 10px 0; border-radius: 4px; }
        .cart-item { background: #f8f9fa; padding: 10px; margin: 5px 0; border-radius: 4px; }
        button { background: #007bff; color: white; padding: 8px 16px; border: none; border-radius: 4px; cursor: pointer; margin: 5px; }
        button:hover { background: #0056b3; }
        .danger { background: #dc3545; }
        .danger:hover { background: #c82333; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Dashboard</h1>
        <div>
            Welcome, <strong><?= htmlspecialchars($username) ?></strong> | 
            <a href="/logout.php">Logout</a>
        </div>
    </div>
    
    <?php if (flash('success')): ?>
        <div class="alert success"><?= htmlspecialchars(flash('success')) ?></div>
    <?php endif; ?>
    
    <?php if (flash('info')): ?>
        <div class="alert info"><?= htmlspecialchars(flash('info')) ?></div>
    <?php endif; ?>
    
    <h2>Session Information</h2>
    <p><strong>Login Time:</strong> <?= date('Y-m-d H:i:s', $loginTime) ?></p>
    <p><strong>Session ID:</strong> <?= session()->getId() ?></p>
    
    <h2>Shopping Cart Demo</h2>
    <div class="product">
        <h3>Product 1 - Laptop</h3>
        <form method="POST" style="display: inline;">
            <?= csrf_field() ?>
            <input type="hidden" name="product_id" value="laptop">
            <button type="submit" name="add_to_cart">Add to Cart</button>
        </form>
    </div>
    
    <div class="product">
        <h3>Product 2 - Mouse</h3>
        <form method="POST" style="display: inline;">
            <?= csrf_field() ?>
            <input type="hidden" name="product_id" value="mouse">
            <button type="submit" name="add_to_cart">Add to Cart</button>
        </form>
    </div>
    
    <h3>Your Cart (<?= array_sum($cart) ?> items)</h3>
    <?php if (empty($cart)): ?>
        <p>Your cart is empty.</p>
    <?php else: ?>
        <?php foreach ($cart as $product => $quantity): ?>
            <div class="cart-item">
                <?= htmlspecialchars(ucfirst($product)) ?> - Quantity: <?= $quantity ?>
            </div>
        <?php endforeach; ?>
        
        <form method="POST" style="margin-top: 10px;">
            <?= csrf_field() ?>
            <button type="submit" name="clear_cart" class="danger">Clear Cart</button>
        </form>
    <?php endif; ?>
    
    <h3>All Session Data</h3>
    <pre><?= htmlspecialchars(print_r(session()->all(), true)) ?></pre>
</body>
</html>
```

### public/logout.php
```php
<?php
// Store username for goodbye message
$username = session('username');

// Clear all session data
session()->flush();

// Regenerate session ID for security
session()->regenerate();

// Set goodbye message
if ($username) {
    flash('info', "Goodbye, {$username}! You have been logged out successfully.");
} else {
    flash('info', 'You have been logged out successfully.');
}

// Redirect to home
header('Location: /');
exit;
?>
```

## 🚀 Setup Instructions

1. **Create the directory structure** as shown above
2. **Set up database** (if using DatabaseSessionManager):
   ```sql
   CREATE DATABASE your_app;
   ```
3. **Configure your web server** to point to the `public/` directory
4. **Update database config** in `config/database.php`
5. **Test the application**:
   - Visit `/` to see the home page
   - Login with `admin/password123` or `user/secret456`
   - Test the shopping cart functionality
   - Check session persistence across page refreshes

This complete example demonstrates Laravel-style session management in a barebone PHP application with all the features you'd expect from a professional framework!