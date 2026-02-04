# Test Scenario Prompt Patterns

Simple, human-like test scenarios. ProxyUser's AI browses like a human—trust it to figure out the details.

## Authentication

User can log in with {{EMAIL}} and {{PASSWORD}}
User sees error with wrong password
User can log out and return to login page
User can request a password reset
User can sign up with a valid email address

## CRUD Operations

User can create a new item
User can view item details
User can edit an existing item
User can delete an item
User can undo a deleted item

## Form Validation

User sees error when required field is empty
User sees error when email format is invalid
User sees password strength warning for weak passwords
User sees success message after valid form submission
User can complete a multi-step form

## Navigation

User can navigate to main sections via menu
User can use breadcrumbs to go back
User can switch between tabs on a page
User can use mobile menu on small screens

## Search and Filtering

User can search and see matching results
User sees "no results" message for invalid search
User can filter by category
User can clear all filters
User can sort results by different criteria

## Pagination

User can load more items
User sees more items when scrolling to bottom
User can navigate between pages

## Modals and Dialogs

User can open a modal
User can close modal with X button
User can close modal with escape key
User sees confirmation dialog before destructive action

## File Upload

User can upload a file
User can drag and drop a file to upload
User sees error for invalid file type

## Notifications

User sees success toast after saving
User sees error banner when something fails

## E-commerce

User can add item to cart
User can update cart quantity
User can remove item from cart
User can apply a discount code
User can proceed through checkout flow

## Settings

User can toggle a setting on and off
User can switch between light and dark mode
User can update profile information

## Tips

- Focus on user intent, not mechanics
- Start with "User can..." or "User sees..."
- One scenario = one capability
- Use {{EMAIL}} and {{PASSWORD}} for credentials
- Only be specific when testing exact text matters
