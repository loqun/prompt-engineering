# Practical Session Implementation

## Login System Example
```php
<?php
session_start();

// Login process
if ($_POST['login']) {
    $username = $_POST['username'];
    $password = $_POST['password'];
    
    if (authenticate($username, $password)) {
        $_SESSION['user_id'] = $user_id;
        $_SESSION['logged_in'] = true;
        header('Location: dashboard.php');
    }
}

// Check if logged in
function isLoggedIn() {
    return isset($_SESSION['logged_in']) && $_SESSION['logged_in'] === true;
}
?>
```

## Shopping Cart Example
```php
<?php
session_start();

// Add item to cart
function addToCart($product_id, $quantity) {
    if (!isset($_SESSION['cart'])) {
        $_SESSION['cart'] = [];
    }
    $_SESSION['cart'][$product_id] = $quantity;
}

// Get cart contents
function getCart() {
    return $_SESSION['cart'] ?? [];
}
?>
```

## Session Security Basics
```php
<?php
// Regenerate session ID for security
session_regenerate_id(true);

// Destroy session on logout
session_destroy();

// Set secure session settings
ini_set('session.cookie_httponly', 1);
ini_set('session.cookie_secure', 1);
?>
```