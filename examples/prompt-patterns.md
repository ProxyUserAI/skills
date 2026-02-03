# Test Scenario Prompt Patterns

A library of effective test scenario prompts for common web application patterns.

## Authentication

### Login with credentials

```
User enters {{EMAIL}} in email field, enters {{PASSWORD}} in password field,
clicks Sign In button, is redirected to dashboard, sees "Welcome" heading
```

### Login with invalid credentials

```
User enters "invalid@test.com" in email field, enters "wrongpassword" in password field,
clicks Sign In button, sees error message "Invalid email or password"
```

### Logout

```
User clicks profile menu in top right, clicks "Log out" option,
is redirected to login page, sees "Sign In" heading
```

### Password reset request

```
User clicks "Forgot password?" link, enters {{EMAIL}} in email field,
clicks Submit, sees message "Check your email for reset instructions"
```

### Sign up flow

```
User clicks "Sign up" link, enters "newuser@test.com" in email field,
enters "SecurePass123!" in password field, enters same in confirm password field,
clicks Create Account, sees welcome screen with "Complete your profile" heading
```

---

## CRUD Operations

### Create item

```
User clicks "New Item" button, enters "Test Item" in name field,
enters "This is a test description" in description field,
clicks Save, sees "Test Item" appear in the items list
```

### Read/View item

```
User clicks on "Widget A" in the items list,
sees detail view with title "Widget A" and description visible
```

### Update item

```
User clicks edit button on "Widget A", changes name to "Widget A (Updated)",
clicks Save, sees updated name "Widget A (Updated)" in the list
```

### Delete item

```
User clicks delete button on first item in list,
confirmation dialog appears asking "Are you sure?",
clicks Confirm, item is removed from list
```

### Delete with undo

```
User clicks delete button on "Test Item",
item disappears from list, sees "Undo" toast notification,
clicks Undo, "Test Item" reappears in list
```

---

## Form Validation

### Required field validation

```
User leaves email field empty, clicks Submit button,
sees error message "Email is required" below the email field
```

### Email format validation

```
User enters "notanemail" in email field, clicks Submit,
sees error message "Please enter a valid email address"
```

### Password strength validation

```
User enters "weak" in password field,
sees warning "Password must be at least 8 characters with one number"
```

### Form success state

```
User fills all required fields with valid data, clicks Submit,
sees success message "Your changes have been saved",
form fields are cleared or disabled
```

### Multi-step form

```
User fills name and email on step 1, clicks Next,
sees step 2 with address fields, fills address, clicks Next,
sees step 3 review page showing all entered information
```

---

## Navigation

### Main navigation

```
User clicks "Products" in main navigation menu,
page loads showing product grid, URL contains "/products"
```

### Breadcrumb navigation

```
User navigates to product detail page,
clicks "Products" in breadcrumb trail,
returns to products list page
```

### Tab navigation

```
User is on account settings page, clicks "Security" tab,
sees password change form and two-factor authentication options
```

### Mobile menu

```
User views page on mobile viewport (375px wide),
clicks hamburger menu icon, sees navigation drawer slide in,
clicks "Dashboard" link, drawer closes and dashboard loads
```

---

## Search and Filtering

### Basic search

```
User types "widget" in search box, presses Enter,
results update to show only items containing "widget" in title or description
```

### Search with no results

```
User types "xyznonexistent" in search box, presses Enter,
sees "No results found" message with suggestion to try different terms
```

### Filter by category

```
User clicks "Category" dropdown, selects "Electronics",
list updates to show only items in Electronics category,
filter chip shows "Electronics" with X to remove
```

### Clear all filters

```
User has multiple filters applied, clicks "Clear all" button,
all filters are removed, full unfiltered list is displayed
```

### Sort results

```
User clicks "Sort by" dropdown, selects "Price: Low to High",
items reorder with lowest price first
```

---

## Pagination

### Load more

```
User sees 10 items in list, scrolls to bottom,
clicks "Load More" button, additional items appear below existing items,
total count increases
```

### Infinite scroll

```
User scrolls to bottom of items list,
loading spinner appears briefly,
10 more items automatically load and appear
```

### Page navigation

```
User is on page 1 of results, clicks "2" in pagination,
page 2 of results loads, page 2 is highlighted in pagination
```

---

## Modals and Dialogs

### Open modal

```
User clicks "Add New" button,
modal overlay appears with form inside,
background content is dimmed
```

### Close modal with X

```
User has modal open, clicks X button in top right,
modal closes, returns to previous view
```

### Close modal with escape

```
User has modal open, presses Escape key,
modal closes
```

### Confirmation dialog

```
User clicks "Delete Account" button,
confirmation dialog appears with "This action cannot be undone" warning,
Cancel and Delete buttons visible
```

---

## File Upload

### Single file upload

```
User clicks "Upload" button, selects image file,
sees file name and preview thumbnail,
clicks Submit, sees "Upload complete" message
```

### Drag and drop upload

```
User drags image file onto upload zone,
zone highlights to indicate drop target,
drops file, sees upload progress bar, then preview
```

### Invalid file type

```
User tries to upload .exe file,
sees error "Only image files (jpg, png, gif) are allowed"
```

---

## Notifications

### Success toast

```
User saves form successfully,
green toast notification appears in top right saying "Saved successfully",
toast disappears after 3 seconds
```

### Error notification

```
User submits form with server error,
red banner appears at top of page with error message,
banner has dismiss button
```

---

## E-commerce

### Add to cart

```
User clicks "Add to Cart" on product page,
cart icon in header updates to show "1",
sees "Added to cart" confirmation
```

### Update cart quantity

```
User is on cart page, changes quantity from 1 to 3 for first item,
line item total and cart total update automatically
```

### Remove from cart

```
User clicks remove button on cart item,
item is removed from cart, cart total updates
```

### Apply discount code

```
User enters "SAVE20" in promo code field, clicks Apply,
sees "20% discount applied" message,
cart total reduces by 20%
```

### Checkout flow

```
User clicks "Checkout" button from cart,
sees shipping address form, fills required fields,
clicks Continue to Payment, sees payment form
```

---

## Settings

### Toggle setting

```
User clicks toggle for "Email notifications",
toggle switches from off to on,
sees "Settings saved" confirmation
```

### Change theme

```
User clicks "Dark mode" toggle,
page theme changes to dark colors immediately
```

### Update profile

```
User changes display name to "New Name", clicks Save,
sees "Profile updated" message,
name in header updates to "New Name"
```

---

## Tips for Writing Prompts

1. **Be specific about elements**: "email field" not just "the field"
2. **Include observable outcomes**: "sees confirmation message" not "form submits"
3. **Use exact text when possible**: 'sees "Welcome back"' not "sees welcome message"
4. **Chain actions clearly**: "clicks X, then clicks Y" or "clicks X, sees Z, clicks Y"
5. **Use environment variables for credentials**: `{{EMAIL}}` not actual values
