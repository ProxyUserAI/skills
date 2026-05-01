# Scenario Prompt Patterns

Simple, human-like scenarios. ProxyUser's AI browses like a human — trust it to figure out the details.

## Authentication

User can log in with {{EMAIL}} and {{PASSWORD}}
User sees error with wrong password
User can log out and return to login page
User can request a password reset
User can sign up with a valid email address

### Authentication Best Practices

**Folder-level authentication** (recommended for most cases):

Set `instructions` on the folder, then keep scenarios simple. All scenarios in the folder inherit the login step.

```
📁 Account Settings
   instructions: "First, log in with {{EMAIL}} and {{PASSWORD}}"

   - User can change display name
   - User can update email preferences
   - User can enable two-factor authentication
```

**Scenario-level authentication** (when the login flow is what you're checking):

```
User can log in with {{EMAIL}} and {{PASSWORD}}
User sees error when entering wrong password
User can log out and return to home page
```

**Starting URL**: When authentication is required, set the starting URL to the login page:
- `https://example.com/login` (correct)
- `https://example.com/` (may redirect unexpectedly)

**Good authentication scenarios:**

```
User can log in with {{EMAIL}} and {{PASSWORD}}
User can log in and access the dashboard
User sees error when password is incorrect
User is redirected to login when accessing protected page
```

**Authentication scenarios to avoid:**

```
User receives login verification email ❌
User enters SMS verification code ❌
User completes Google OAuth flow ❌
```

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

## Verification-Only Scenarios

For pages where the goal is just *"this content should be visible"* — no clicks, no forms — use plain assertion phrasing. The planner verifies the text on the rendered page.

```
Pricing page shows Starter, Pro, and Scale tiers with monthly prices
Homepage hero shows headline "Reply faster, sound human"
BotBlock feature page explains bot detection for X replies
Privacy policy page lists data retention details
Changelog page shows the latest release notes
```

**Pick distinctive phrases.** Avoid asserting page chrome ("Sign up", "Login", "Footer") — those appear everywhere and don't prove the goal. Pick a 5–10 word substring that's specific to this page's content.

**When to use these instead of an action scenario:**

- Marketing pages where you ship copy changes (homepage, features, pricing)
- Legal/compliance pages where the *content* is the contract (privacy, terms)
- Documentation pages where the goal is "the page exists and explains X"
- Status banners or product-update announcements

If the goal involves a user *doing* something (clicking, signing up, searching), write an action scenario instead.

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
- Start with "User can..." or "User sees..." for action scenarios; use "Page shows…" / "Page displays…" for verification-only scenarios
- One scenario = one capability
- Use {{EMAIL}} and {{PASSWORD}} for credentials
- Only be specific when exact text matters

## Scenarios to Avoid

These require actions outside the browser—ProxyUser can't complete them:

- User clicks verification link in email ❌
- User enters SMS code ❌
- User completes Google/Apple/Facebook login ❌
- User receives Slack/Discord notification ❌

See SKILL.md "Scenario Boundaries" for alternatives.
