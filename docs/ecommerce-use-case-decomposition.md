# E-Commerce Web Application – Use Case Decomposition

Companion to [ecommerce-use-cases.md](./ecommerce-use-cases.md) and [ecommerce-use-case-diagrams.md](./ecommerce-use-case-diagrams.md). Each diagram breaks a use case into sub-use-cases (functions), top-down.

**Legend**
- Rectangle at the top = the use case being decomposed.
- Solid arrow = mandatory sub-use-case (**include** / part of the main flow).
- Dashed arrow labelled `extend` = optional or conditional sub-use-case (alternative flow).
- Rounded node = an external actor/system or another use case referenced (e.g. `UC-29`).
- Leaf nodes are the smallest steps to implement.

---

## 1. Customer-facing use cases

### UC-01 Register Account
```mermaid
flowchart TD
    UC["UC-01 Register Account"]
    UC --> A["1.1 Enter registration data"]
    UC --> B["1.2 Validate input"]
    UC --> C["1.3 Create unverified account"]
    UC --> D["1.4 Verify email"]
    UC --> E["1.5 Activate account and log in"]
    A --> A1["Name, email, password"]
    A --> A2["Accept terms"]
    B --> B1["Email format"]
    B --> B2["Password strength"]
    B --> B3["Unique email check"]
    C --> C1["Hash password"]
    C --> C2["Store account record"]
    D --> D1["Generate token, valid 24h"]
    D --> D2(["UC-29 Send verification email"])
    D -.->|extend| D3["Resend verification email"]
    E --> E1["Merge guest cart"]
    UC -.->|extend| S["Social sign-up via OAuth"]
    S --> S1["Obtain OAuth consent"]
    S --> S2["Create verified account"]
```

### UC-02 Log In / Log Out
```mermaid
flowchart TD
    UC["UC-02 Log In / Log Out"]
    UC --> A["2.1 Authenticate credentials"]
    UC --> B["2.2 Create session / JWT"]
    UC --> C["2.3 Merge guest cart"]
    UC --> D["2.4 Redirect to previous page"]
    UC --> E["2.5 Log out"]
    A --> A1["Check email and password hash"]
    A --> A2["Count failed attempts"]
    A -.->|extend| F["Lockout / CAPTCHA after 5 failures"]
    A -.->|extend| G["Verify MFA code"]
    E --> E1["Invalidate session / token"]
```

### UC-03 Reset Password
```mermaid
flowchart TD
    UC["UC-03 Reset Password"]
    UC --> A["3.1 Request reset"]
    UC --> B["3.2 Issue reset link"]
    UC --> C["3.3 Set new password"]
    UC --> D["3.4 Post-reset security actions"]
    A --> A1["Enter email"]
    A --> A2["Show identical message always"]
    B --> B1["Generate time-limited token"]
    B --> B2(["UC-29 Send reset email"])
    B -.->|extend| B3["Request new link if expired or used"]
    C --> C1["Validate token"]
    C --> C2["Validate password strength"]
    C --> C3["Hash and save password"]
    D --> D1["Invalidate all sessions"]
    D --> D2(["UC-29 Send confirmation email"])
```

### UC-04 Manage Profile & Addresses
```mermaid
flowchart TD
    UC["UC-04 Manage Profile and Addresses"]
    UC --> A["4.1 Edit personal info"]
    UC --> B["4.2 Manage addresses"]
    UC --> C["4.3 Change password"]
    UC --> D["4.4 Manage saved payment methods"]
    B --> B1["Add address"]
    B --> B2["Edit address"]
    B --> B3["Delete address"]
    B --> B4["Validate address"]
    B -.->|extend| B5["Set default shipping / billing"]
    C --> C1["Verify current password"]
    C --> C2["Save new password"]
    D --> D1["Tokenize card via gateway"]
    D --> D2["Remove saved method"]
    D --> D3(["Payment Gateway"])
```

### UC-05 Browse Catalog by Category
```mermaid
flowchart TD
    UC["UC-05 Browse Catalog by Category"]
    UC --> A["5.1 Load category products"]
    UC --> B["5.2 Sort products"]
    UC --> C["5.3 Paginate"]
    UC --> D["5.4 Navigate sub-categories"]
    A -.->|extend| A1["Show empty state + recommendations"]
    B --> B1["Price"]
    B --> B2["Newest"]
    B --> B3["Popularity"]
    D --> D1["Breadcrumbs"]
    D --> D2["Sub-category menu"]
    UC --> E(["UC-07 View product details"])
```

### UC-06 Search & Filter Products
```mermaid
flowchart TD
    UC["UC-06 Search and Filter Products"]
    UC --> A["6.1 Enter query"]
    UC --> B["6.2 Run search"]
    UC --> C["6.3 Apply filters"]
    UC --> D["6.4 Show results"]
    A --> A1["Autocomplete suggestions"]
    B --> B1["Typo tolerance"]
    B --> B2(["Search index"])
    C --> C1["Price range"]
    C --> C2["Brand"]
    C --> C3["Rating"]
    C --> C4["Availability"]
    C --> C5["Attributes"]
    D -.->|extend| D1["No results: did you mean + popular products"]
```

### UC-07 View Product Details
```mermaid
flowchart TD
    UC["UC-07 View Product Details"]
    UC --> A["7.1 Show product info"]
    UC --> B["7.2 Select variant"]
    UC --> C["7.3 Show stock status"]
    UC --> D["7.4 Show social proof and related"]
    A --> A1["Image gallery"]
    A --> A2["Title, price"]
    A --> A3["Description, specs"]
    A --> A4["Shipping estimate"]
    B --> B1["Size / colour options"]
    B -.->|extend| B2["Disable unavailable variant"]
    C -.->|extend| C1["Notify me when available"]
    D --> D1["Reviews and ratings"]
    D --> D2["Related products"]
    UC --> E(["UC-08 Add to cart"])
    UC --> F(["UC-09 Add to wishlist"])
```

### UC-08 Manage Shopping Cart
```mermaid
flowchart TD
    UC["UC-08 Manage Shopping Cart"]
    UC --> A["8.1 Add item"]
    UC --> B["8.2 View cart"]
    UC --> C["8.3 Update cart"]
    UC --> D["8.4 Recalculate totals"]
    UC --> E["8.5 Persist cart"]
    A --> A1["Select variant and quantity"]
    A --> A2["Validate stock"]
    A -.->|extend| A3["Cap quantity to stock"]
    C --> C1["Change quantity"]
    C --> C2["Remove item"]
    C --> C3["Save for later"]
    D --> D1["Subtotal"]
    D --> D2["Discounts"]
    D --> D3["Estimated tax and shipping"]
    D -.->|extend| D4["Price-change notice"]
    E --> E1["Guest: cookie"]
    E --> E2["Customer: database"]
    UC -.->|extend| F(["UC-10 Apply coupon"])
    UC -.->|extend| G(["UC-11 Checkout"])
```

### UC-09 Manage Wishlist
```mermaid
flowchart TD
    UC["UC-09 Manage Wishlist"]
    UC --> A["9.1 Add product"]
    UC --> B["9.2 Remove product"]
    UC --> C["9.3 View wishlist"]
    UC --> D["9.4 Move to cart"]
    UC --> E["9.5 Share list"]
    A --> A1(["UC-02 Log in required"])
    D --> D1(["UC-08 Manage cart"])
    E --> E1["Generate share link"]
```

### UC-10 Apply Coupon / Discount Code
```mermaid
flowchart TD
    UC["UC-10 Apply Coupon"]
    UC --> A["10.1 Enter code"]
    UC --> B["10.2 Validate coupon"]
    UC --> C["10.3 Resolve promo conflicts"]
    UC --> D["10.4 Apply discount"]
    B --> B1["Active"]
    B --> B2["Date range"]
    B --> B3["Minimum spend"]
    B --> B4["Usage limit"]
    B --> B5["Eligible items"]
    B -.->|extend| B6["Show specific error"]
    C -.->|extend| C1["Keep best discount or reject"]
    D --> D1["Update cart total"]
    D --> D2["Display discount line"]
```

### UC-11 Checkout & Place Order
```mermaid
flowchart TD
    UC["UC-11 Checkout and Place Order"]
    UC --> A["11.1 Provide shipping address"]
    UC --> B["11.2 Choose shipping method"]
    UC --> C["11.3 Calculate tax and total"]
    UC --> D["11.4 Select payment method and review"]
    UC --> E["11.5 Place order"]
    A --> A1["Select saved or enter new address"]
    A --> A2["Guest email"]
    A -.->|extend| A3["Address not serviceable"]
    B --> B1(["Shipping Provider: rates"])
    E --> E1["Re-validate stock and prices"]
    E --> E2(["UC-30 Reserve stock"])
    E --> E3["Create order: Pending Payment"]
    E -.->|extend| E4["Remove unavailable item"]
    UC --> F(["UC-12 Pay for order"])
```

### UC-12 Pay for Order
```mermaid
flowchart TD
    UC["UC-12 Pay for Order"]
    UC --> A["12.1 Create payment intent"]
    UC --> B["12.2 Capture payment details"]
    UC --> C["12.3 Confirm result"]
    UC --> D["12.4 Finalize order"]
    A --> A1(["Payment Gateway"])
    B --> B1["Hosted payment fields"]
    B -.->|extend| B2["Cash on delivery: order Confirmed"]
    C --> C1(["UC-31 Handle webhook"])
    C -.->|extend| C2["Declined: retry or other method"]
    C -.->|extend| C3["Timeout: expire order"]
    C3 --> C4(["UC-30 Release stock"])
    D --> D1["Order status Paid"]
    D --> D2["Show confirmation page"]
    D --> D3(["UC-29 Send confirmation email"])
```

### UC-13 Track Order
```mermaid
flowchart TD
    UC["UC-13 Track Order"]
    UC --> A["13.1 Identify order"]
    UC --> B["13.2 Show status timeline"]
    UC --> C["13.3 Show carrier tracking"]
    A --> A1["Customer: select from account"]
    A -.->|extend| A2["Guest: order number + email"]
    B --> B1["Paid"]
    B --> B2["Packed"]
    B --> B3["Shipped"]
    B --> B4["Out for delivery"]
    B --> B5["Delivered"]
    C --> C1["Tracking number and link"]
    C --> C2(["Shipping Provider: events"])
```

### UC-14 View Order History
```mermaid
flowchart TD
    UC["UC-14 View Order History"]
    UC --> A["14.1 List orders"]
    UC --> B["14.2 Filter orders"]
    UC --> C["14.3 View order detail"]
    A --> A1["Pagination"]
    B --> B1["By date"]
    B --> B2["By status"]
    C --> C1["Items, totals, addresses"]
    C -.->|extend| C2["Download invoice"]
    C -.->|extend| C3["Buy again"]
    C -.->|extend| C4(["UC-13 Track"])
    C -.->|extend| C5(["UC-15 Cancel / UC-16 Return"])
```

### UC-15 Cancel Order
```mermaid
flowchart TD
    UC["UC-15 Cancel Order"]
    UC --> A["15.1 Submit cancel request"]
    UC --> B["15.2 Check eligibility"]
    UC --> C["15.3 Cancel order"]
    UC --> D["15.4 Notify customer"]
    A --> A1["Enter reason"]
    B -.->|extend| B1(["UC-16 Redirect to return if shipped"])
    C --> C1["Status Cancelled"]
    C --> C2(["UC-30 Release stock"])
    C -.->|extend| C3["Refund if paid"]
    C3 --> C4(["Payment Gateway"])
    D --> D1(["UC-29 Send notification"])
```

### UC-16 Request Return / Refund
```mermaid
flowchart TD
    UC["UC-16 Request Return / Refund"]
    UC --> A["16.1 Submit return request"]
    UC --> B["16.2 Review request"]
    UC --> C["16.3 Issue return label"]
    UC --> D["16.4 Receive and inspect item"]
    UC --> E["16.5 Refund"]
    A --> A1["Select items and quantity"]
    A --> A2["Reason"]
    A -.->|extend| A3["Upload photos"]
    A --> A4["Check return window and eligibility"]
    B --> B1(["Support Agent"])
    B -.->|extend| B2["Reject with reason"]
    D --> D1(["Warehouse"])
    E --> E1["Refund to original method or store credit"]
    E -.->|extend| E2["Partial refund if damaged"]
    E --> E3["Restock if resellable"]
    E --> E4(["Payment Gateway"])
```

### UC-17 Write Product Review & Rating
```mermaid
flowchart TD
    UC["UC-17 Write Review and Rating"]
    UC --> A["17.1 Verify purchase"]
    UC --> B["17.2 Compose review"]
    UC --> C["17.3 Screen content"]
    UC --> D["17.4 Submit for moderation"]
    UC --> E["17.5 Publish"]
    B --> B1["Rating"]
    B --> B2["Title and text"]
    B -.->|extend| B3["Photos"]
    B -.->|extend| B4["Edit existing review"]
    C --> C1["Profanity filter"]
    D --> D1(["UC-25 Moderate reviews"])
    E --> E1["Update average rating"]
```

### UC-18 Contact Customer Support
```mermaid
flowchart TD
    UC["UC-18 Contact Customer Support"]
    UC --> A["18.1 Submit inquiry"]
    UC --> B["18.2 Create ticket"]
    UC --> C["18.3 Handle ticket"]
    UC --> D["18.4 Close ticket"]
    A --> A1["Form or chat"]
    A -.->|extend| A2["Attach order reference"]
    B --> B1["Send acknowledgment"]
    C --> C1(["Support Agent replies"])
    C --> C2["Customer follow-up"]
```

---

## 2. Back-office use cases

### UC-19 Manage Products & Categories
```mermaid
flowchart TD
    UC["UC-19 Manage Products and Categories"]
    UC --> A["19.1 Create / edit product"]
    UC --> B["19.2 Manage categories"]
    UC --> C["19.3 Publish or draft"]
    UC --> D["19.4 Delete / archive product"]
    UC -.->|extend| E["19.5 Bulk import CSV"]
    A --> A1["Name, SKU, description"]
    A --> A2["Images"]
    A --> A3["Price"]
    A --> A4["Variants and attributes"]
    A --> A5["SEO fields"]
    A --> A6["Validate unique SKU"]
    B --> B1["Create / rename / reorder"]
    D -.->|extend| D1["Archive if order history exists"]
    E --> E1["Validate rows"]
    E --> E2["Error report"]
```

### UC-20 Manage Inventory
```mermaid
flowchart TD
    UC["UC-20 Manage Inventory"]
    UC --> A["20.1 Adjust stock"]
    UC --> B["20.2 Log adjustment"]
    UC --> C["20.3 Monitor low stock"]
    A --> A1["Manual per SKU / warehouse"]
    A -.->|extend| A2["Import stock file"]
    A --> A3["Validate no negative stock unless backorder"]
    C --> C1["Compare with threshold"]
    C --> C2(["UC-29 Low-stock alert"])
```

### UC-21 Manage Orders
```mermaid
flowchart TD
    UC["UC-21 Manage Orders"]
    UC --> A["21.1 Search and filter orders"]
    UC --> B["21.2 View order"]
    UC --> C["21.3 Update status"]
    UC --> D["21.4 Fulfil shipment"]
    UC --> E["21.5 Notify customer"]
    C --> C1["Packed"]
    C --> C2["Shipped"]
    D --> D1["Enter or generate tracking number"]
    D --> D2["Print label"]
    D --> D3(["Shipping Provider"])
    D -.->|extend| D4["Split shipment"]
    B -.->|extend| B1["Edit address before shipping"]
    B -.->|extend| B2["Cancel order + refund"]
    E --> E1(["UC-29 Send notification"])
```

### UC-22 Process Returns & Refunds
```mermaid
flowchart TD
    UC["UC-22 Process Returns and Refunds"]
    UC --> A["22.1 Review return requests"]
    UC --> B["22.2 Approve / reject"]
    UC --> C["22.3 Confirm receipt"]
    UC --> D["22.4 Issue refund"]
    UC --> E["22.5 Record in audit log"]
    D --> D1["Full refund"]
    D -.->|extend| D2["Partial refund"]
    D --> D3(["Payment Gateway"])
    D -.->|extend| D4["Flag failed refund for manual handling"]
```

### UC-23 Manage Customers
```mermaid
flowchart TD
    UC["UC-23 Manage Customers"]
    UC --> A["23.1 Search customers"]
    UC --> B["23.2 View profile and orders"]
    UC --> C["23.3 Change account status"]
    UC --> D["23.4 Handle GDPR request"]
    C --> C1["Deactivate"]
    C --> C2["Reactivate"]
    D --> D1["Anonymize personal data"]
    D -.->|extend| D2["Export customer data"]
```

### UC-24 Manage Promotions & Coupons
```mermaid
flowchart TD
    UC["UC-24 Manage Promotions and Coupons"]
    UC --> A["24.1 Define promotion type"]
    UC --> B["24.2 Set conditions"]
    UC --> C["24.3 Set validity and limits"]
    UC --> D["24.4 Activate / deactivate"]
    A --> A1["Percentage"]
    A --> A2["Fixed amount"]
    A --> A3["Free shipping"]
    A --> A4["BOGO"]
    B --> B1["Minimum spend"]
    B --> B2["Eligible products / categories"]
    C --> C1["Start and end dates"]
    C --> C2["Usage limits"]
    D --> D1["Automatic rule"]
    D -.->|extend| D2["Generate coupon codes"]
```

### UC-25 Moderate Reviews
```mermaid
flowchart TD
    UC["UC-25 Moderate Reviews"]
    UC --> A["25.1 List pending / flagged reviews"]
    UC --> B["25.2 Decide"]
    B --> B1["Approve"]
    B --> B2["Reject"]
    B --> B3["Delete"]
    B1 --> C["Update average rating"]
    B2 --> D(["UC-29 Notify author"])
```

### UC-26 View Sales Reports & Analytics
```mermaid
flowchart TD
    UC["UC-26 View Reports and Analytics"]
    UC --> A["26.1 Select date range"]
    UC --> B["26.2 Generate metrics"]
    UC --> C["26.3 Display dashboard"]
    UC -.->|extend| D["26.4 Export CSV / PDF"]
    B --> B1["Revenue and orders"]
    B --> B2["Average order value"]
    B --> B3["Conversion"]
    B --> B4["Top products"]
    B --> B5["Stock value"]
    B --> B6["Refunds"]
```

### UC-27 Configure Shipping, Tax & Payment Settings
```mermaid
flowchart TD
    UC["UC-27 Configure Settings"]
    UC --> A["27.1 Shipping settings"]
    UC --> B["27.2 Tax settings"]
    UC --> C["27.3 Payment settings"]
    UC --> D["27.4 Store information"]
    UC --> E["27.5 Email templates"]
    A --> A1["Zones"]
    A --> A2["Rates"]
    B --> B1["Tax rules per region"]
    C --> C1["Enable payment methods"]
    C --> C2["Gateway credentials"]
    UC --> F["27.6 Save and audit log"]
```

### UC-28 Manage Staff Roles & Permissions
```mermaid
flowchart TD
    UC["UC-28 Manage Staff Roles and Permissions"]
    UC --> A["28.1 Manage staff accounts"]
    UC --> B["28.2 Manage roles"]
    UC --> C["28.3 Enforce RBAC"]
    UC --> D["28.4 Audit sensitive actions"]
    A --> A1["Create"]
    A --> A2["Edit / disable"]
    B --> B1["Catalog Manager"]
    B --> B2["Order Manager"]
    B --> B3["Support"]
    B --> B4["Assign permissions"]
    C --> C1["Check permission per request"]
    C -.->|extend| C2["Deny and log"]
```

---

## 3. System / automated use cases

### UC-29 Send Transactional Notifications
```mermaid
flowchart TD
    UC["UC-29 Send Transactional Notifications"]
    UC --> A["29.1 Receive trigger event"]
    UC --> B["29.2 Render template"]
    UC --> C["29.3 Send message"]
    UC --> D["29.4 Log delivery"]
    A --> A1["Registration, order, payment"]
    A --> A2["Shipped, delivered, refund, reset"]
    C --> C1["Email"]
    C --> C2["SMS"]
    C --> C3(["Notification Service"])
    C -.->|extend| C4["Retry with backoff"]
```

### UC-30 Reserve & Release Stock
```mermaid
flowchart TD
    UC["UC-30 Reserve and Release Stock"]
    UC --> A["30.1 Reserve stock"]
    UC --> B["30.2 Commit stock"]
    UC --> C["30.3 Release stock"]
    A --> A1["Atomic availability check"]
    A --> A2["Decrement available quantity"]
    B --> B1["On payment success"]
    C --> C1["On payment expiry"]
    C --> C2["On order cancel"]
    C --> C3["Scheduled expiry sweep"]
```

### UC-31 Handle Payment Webhooks
```mermaid
flowchart TD
    UC["UC-31 Handle Payment Webhooks"]
    UC --> A["31.1 Receive event"]
    UC --> B["31.2 Verify signature"]
    UC --> C["31.3 Deduplicate event"]
    UC --> D["31.4 Update payment and order"]
    UC --> E["31.5 Respond 200"]
    A --> A1(["Payment Gateway"])
    C -.->|extend| C1["Ignore and log unknown / duplicate"]
    D --> D1(["UC-29 Trigger notifications"])
    B -.->|extend| B1["Reject and alert on invalid signature"]
```

### UC-32 Abandoned Cart Reminder
```mermaid
flowchart TD
    UC["UC-32 Abandoned Cart Reminder"]
    UC --> A["32.1 Detect idle carts, 24h"]
    UC --> B["32.2 Check eligibility"]
    UC --> C["32.3 Send reminder"]
    UC --> D["32.4 Stop reminders"]
    B --> B1["Email known"]
    B --> B2["Not purchased"]
    B --> B3["Not unsubscribed"]
    C --> C1(["UC-29 Send notification"])
    C -.->|extend| C2["Include incentive"]
    D --> D1["On purchase"]
    D --> D2["On unsubscribe"]
```
