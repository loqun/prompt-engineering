# HTTP Sessions - The Basics

## What is a Session? (5-Year-Old Explanation)
Imagine you go to a restaurant. The waiter gives you a special number card. Every time you order something, you show this card, and the waiter remembers what you've ordered before. That's exactly how sessions work on websites!

## Real World Problem
HTTP is "stateless" - each request is independent. Without sessions, a website can't remember you logged in on the previous page.

## How Sessions Work
1. **First Visit**: Server creates unique session ID
2. **Storage**: Session ID stored in cookie or URL
3. **Subsequent Requests**: Browser sends session ID back
4. **Server Recognition**: Server uses ID to retrieve your data

## Basic PHP Session Example
```php
<?php
// Start session
session_start();

// Store data
$_SESSION['username'] = 'john';
$_SESSION['role'] = 'admin';

// Retrieve data
echo "Hello " . $_SESSION['username'];
?>
```

## Key Concepts
- **Session ID**: Unique identifier (like your restaurant number)
- **Session Data**: Information stored on server
- **Session Cookie**: How browser remembers the ID
- **Session Lifetime**: How long session stays active