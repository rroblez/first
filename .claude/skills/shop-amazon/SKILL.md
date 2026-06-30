---
name: shop-amazon
description: Browse and purchase items on Amazon.com via Chrome DevTools MCP. Use when the user asks to find, compare, or buy products on Amazon.
scripts: []
---

# Amazon Shopping via Chrome DevTools

## Goal
Search, browse, and purchase products on Amazon.com using the Chrome DevTools MCP server. The agent drives a real Chrome browser to navigate Amazon, find products, add them to cart, and complete checkout — with explicit user approval before any purchase.

## CRITICAL RULES

1. **NEVER place an order without explicit user approval.** Always stop before clicking "Place your order" and confirm with the user. No exceptions.
2. **NEVER store or log payment information, passwords, or personal details.**
3. **NEVER auto-fill payment methods or shipping addresses** — let Amazon's saved defaults handle this.
4. **Always show the user what you're about to buy** (product name, price, quantity) before adding to cart.
5. **Always show the order total** (including tax/shipping) before final purchase confirmation.

## Inputs
- **Search query**: What the user wants to buy (e.g., "RG6 coax cable 6ft")
- **Preferences**: Any filters like price range, brand, Prime-only, ratings, etc.
- **Quantity**: How many to order (default: 1)

## Tools
- **Chrome DevTools MCP** (`mcp__chrome-devtools__*`): All browser interaction
  - `new_page` / `list_pages`: Open/manage browser tabs
  - `navigate_page`: Go to a URL
  - `take_screenshot`: See current page state, or a single product image when given an element `uid`
  - `take_snapshot`: Get the a11y tree (titles, prices, ratings, element `uid`s for clicking/screenshotting)
  - `click` / `type_text`: Interact with page elements

## Process

### 1. Start Chrome / Connect to Existing Session

Check if Chrome is already running:
```
mcp__chrome-devtools__list_pages
```

If it errors with "browser is already running", kill stale processes first:
```bash
pkill -f "chrome-devtools-mcp" || true
pkill -f "Google Chrome.*chrome-devtools-mcp" || true
sleep 1
```

Then open Amazon:
```
mcp__chrome-devtools__new_page → url: "https://www.amazon.com"
```

### 2. Verify Login Status

Take a screenshot to check if the user is logged in:
```
mcp__chrome-devtools__screenshot
```

Look for "Hello, [Name]" in the top nav. If not logged in:
- Navigate to the sign-in page: `https://www.amazon.com/ap/signin`
- **Tell the user to log in manually** — never type credentials yourself
- Wait for user confirmation that they're signed in
- Take another screenshot to verify

### 3. Search for Products

Navigate to search or use the search bar:
```
mcp__chrome-devtools__navigate → url: "https://www.amazon.com/s?k=<search+terms>"
```

Take a snapshot to see results. For each top candidate (typically 3-5), extract the
ASIN from its product link and present a shortened canonical link in the form
`https://www.amazon.com/dp/<ASIN>` — strip tracking/query params, and never substitute
a third-party URL shortener. Don't screenshot every candidate at this stage; text +
links are enough to compare and are much faster than per-product screenshots.

Present each option with:
- Product name
- Shortened link
- Price
- Rating and review count
- Prime eligibility
- Delivery estimate

Ask the user which product they want, or if they want to refine the search.

### 4. View Product Details

Click into the selected product. Take *one* screenshot of this product page (no `uid`
— a full-page screenshot is more reliable than targeting a specific image element,
which can hang on listings with embedded video/gallery widgets) and include it, along
with the shortened `https://www.amazon.com/dp/<ASIN>` link. If the screenshot times
out, retry once; if it times out again, skip it and proceed with text only — don't
keep retrying. Confirm with the user:
- Correct product
- Correct size/color/variant
- Price
- Quantity

### 5. Add to Cart

Only after user confirms the product:
```
Click "Add to cart" button
```

Take a screenshot to confirm it was added. Show the user the cart summary.

### 6. Proceed to Checkout

Navigate to cart: `https://www.amazon.com/gp/cart/view.html`

Take a screenshot. Show the user:
- All items in cart
- Quantities
- Subtotal
- Shipping address (confirm it's correct)
- Delivery estimates

### 7. STOP — Get Purchase Approval

**This is mandatory. Do not skip this step.**

Show the user the complete order summary:
- Items and quantities
- Subtotal
- Shipping cost
- Tax
- **Order total**
- Delivery date

Ask: "Should I place this order for $X.XX total?"

**Only proceed if the user explicitly says yes.**

### 8. Place Order

Click "Place your order" only after explicit approval. Take a screenshot of the order confirmation. Share the order number with the user.

## Outputs (Deliverables)
- **Order confirmation**: Order number and expected delivery date reported to user
- **No local files are created** — this is a browser-only workflow

## Edge Cases

- **Item out of stock**: Inform the user, suggest alternatives or "Notify when available"
- **Price changed since discussion**: Alert the user to the new price before proceeding
- **Multiple sellers**: Default to Amazon-fulfilled/shipped. Flag third-party sellers to user
- **Cart already has items**: Show existing cart contents to user before adding new items — don't clear their cart without asking
- **Shipping address prompt**: If Amazon asks to select an address, take a screenshot and let the user choose — don't select for them
- **CAPTCHA or verification**: Take a screenshot, ask user to solve it manually
- **Session expired mid-flow**: Ask user to log in again, restart from step 2

## Error Handling

- **Chrome not responding**: Kill processes (`pkill -f chrome-devtools-mcp`), restart, re-navigate
- **Page not loading**: Wait 3 seconds, retry navigation once. If still failing, inform user
- **Element not found (button, field)**: Take a screenshot to assess page state. Try scrolling down. If still not found, ask user for guidance
- **MCP connection error**: Restart the MCP server, reconnect

## Notes

- Amazon.com is the target site (user is in Alameda, Ca)
- User has Prime (look for Prime delivery options)
- The user's Chrome profile at `~/.cache/chrome-devtools-mcp/chrome-profile` may retain login sessions between uses
- Screenshot at decision points that need visual confirmation (product chosen, cart, checkout, order placed) — not on every step. If a screenshot call times out twice in a row, stop retrying and continue with text/snapshot data instead.
- This workflow uses NO execution scripts — it's entirely browser-driven via MCP
- When comparing products for the user, focus on: price per unit, review count + rating, Prime eligibility, and delivery speed