# Practice Exercises

## Exercise 1: Basic Login System
Create a simple login form that:
- Stores username in session after login
- Shows "Welcome [username]" on success
- Provides logout functionality

## Exercise 2: Shopping Cart
Build a cart system that:
- Adds products to session-based cart
- Displays cart contents
- Calculates total price
- Persists across page refreshes

## Exercise 3: User Preferences
Create a system to:
- Store user theme preference (dark/light)
- Remember language selection
- Save last visited page

## Exercise 4: Session Security
Implement:
- Session timeout after 15 minutes
- Session regeneration on login
- Secure cookie settings

## Common Debugging Tips
```php
// Debug session contents
echo '<pre>';
print_r($_SESSION);
echo '</pre>';

// Check session status
echo 'Session Status: ' . session_status();
echo 'Session ID: ' . session_id();
```

## Testing Checklist
- [ ] Sessions work across multiple pages
- [ ] Data persists after browser refresh
- [ ] Logout properly destroys session
- [ ] Session expires after timeout
- [ ] No session data leaks between users