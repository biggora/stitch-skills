# Prompting Google Stitch for screen generation

A reference for writing prompts passed to `generate_screen_from_text` (and, by extension, `edit_screens` / `generate_variants` in `stitch-iterate`). Read this before writing any prompt beyond a one-line placeholder.

## The anatomy of an effective Stitch prompt

Structure a non-trivial screen prompt as a single flowing paragraph (not a bulleted list — Stitch reads it as a natural-language brief) that moves through these six elements in order:

1. **Screen purpose.** One sentence naming what the screen is for and who uses it. ("A settings screen for a fitness tracking app where users manage their account and notification preferences.")
2. **Layout structure.** The overall skeleton: header/nav placement, primary content area, secondary panels, footer, tab bar. Name the arrangement, not just the pieces ("a fixed top app bar, a scrollable list body, and a floating action button in the bottom-right").
3. **Named regions/components.** Call out each concrete UI component by name and where it sits: cards, tables, form fields, buttons, chips, avatars, charts. Vague nouns like "some info" or "the usual controls" leave Stitch to guess.
4. **Content specifics.** Real or realistic sample content — actual field labels, actual button text, actual placeholder numbers or names — not "appropriate content" or "relevant fields." Generic placeholders produce generic screens.
5. **Interaction states.** Call out any state the screen must visibly represent: empty, loading, error, selected/active, disabled, success. If a state matters, say so explicitly — Stitch will not infer that you care about an error state unless you ask for it.
6. **Explicit exclusions.** Say what should NOT appear if there's a reasonable chance Stitch adds it unprompted — e.g. "no search bar," "do not include a footer," "skip the marketing banner." Exclusions are cheap insurance against the model over-decorating the screen.

A prompt that only hits purpose and a vague layout ("a settings page with some options") will generate *something*, but it leaves every other decision to chance. The more of the six elements you specify, the closer the first generation lands to what's actually wanted — and the fewer expensive `edit_screens` round trips are needed afterward.

## Weak vs. strong: five comparisons

| Weak prompt | What's missing | Strong prompt |
|---|---|---|
| "A login page" | Layout, fields, states, content | "A mobile login screen for a productivity app with a centered card containing an email field, a password field with a show/hide toggle, a primary 'Log In' button, a 'Forgot password?' link below it, and a secondary 'Create an account' link at the bottom. Include a visible inline error state under the password field reading 'Incorrect email or password.' No social login buttons." |
| "A dashboard" | Named widgets, data specifics, layout | "A desktop analytics dashboard with a left sidebar for navigation (Dashboard, Reports, Settings), a top row of four metric cards (Total Revenue, Active Users, Conversion Rate, Avg. Session), and a main area with a line chart of revenue over the last 30 days and a table of the 5 most recent orders with columns for Order ID, Customer, Amount, and Status." |
| "A list of items with filters" | What's listed, what filters, empty state | "A mobile screen listing job applications as a vertical list of cards, each showing candidate name, applied role, and a status chip (New, Interviewing, Rejected, Hired). Above the list, a horizontal row of filter chips for each status plus an 'All' chip. Include an empty state shown when a filter has zero matches, with the message 'No candidates match this filter.'" |
| "A detail page" | Which entity, which fields, actions | "A desktop order detail screen showing an order header (Order #48213, placed March 2, status badge 'Shipped'), a two-column layout with shipping and billing address on the left and a line-item table (product, quantity, unit price, subtotal) on the right, and a summary panel with subtotal, tax, shipping, and total. Include 'Refund Order' and 'Print Invoice' buttons in the header." |
| "A checkout form" | Steps, fields per step, progress indicator | "A mobile multi-step checkout flow shown as step 2 of 3 (Shipping → Payment → Review), with a progress indicator at the top. This step shows a payment form with card number, expiry, CVV, and cardholder name fields, a toggle for 'Billing address same as shipping,' and 'Back' and 'Continue to Review' buttons pinned to the bottom." |

## Eight ready-to-adapt example prompts

Each of these is copy-ready as a starting point for `prompt` in `generate_screen_from_text` — adjust names, numbers, and specific copy to the actual product.

**Search results with facets**
> A mobile search results screen for an online furniture marketplace, reached after searching 'outdoor dining table'. A search bar pinned at the top pre-filled with 'outdoor dining table' and a filter icon button beside it. Below it, a horizontal scrollable row of facet chips: Price, Material, Seats, Brand. The main body is a two-column grid of product cards, each with a photo, name, price, and star rating (e.g. '4.6 (312)'). Show a loading skeleton state for the grid while results are fetching. Include a zero-results state with the message 'No results for "outdoor dining table" with these filters' and a 'Clear all filters' button. No sort-by dropdown.

**Chat / messaging thread**
> A mobile messaging thread screen between the user and a contact named 'Priya Shah', for a team chat app. A top bar showing Priya's avatar, name, and a status label 'Active now'. The body is a scrollable message list with received messages left-aligned in gray bubbles and sent messages right-aligned in blue bubbles, each with a timestamp below it (e.g. '2:14 PM'). Show the most recent sent message in a pending-send state with a small gray clock icon instead of a delivered checkmark. At the bottom, a fixed input bar with a text field placeholder 'Message Priya...', an attachment icon, and a send button. No typing indicator.

**Calendar / scheduling view**
> A desktop scheduling screen for a doctor's office booking tool, showing a weekly calendar grid for the week of March 9–15. Days appear as columns (Mon–Sun) and half-hour time slots as rows from 8:00 AM to 6:00 PM. Existing appointments appear as colored blocks labeled with patient name and appointment type (e.g. 'J. Torres — Checkup, 9:00–9:30'). A left sidebar lists providers with checkboxes to toggle their schedules on the grid. Show the slot at Wednesday 2:00 PM in a selected state with a highlighted blue outline and a floating 'Book Appointment' button anchored to it. No month-view toggle.

**User profile (view vs. edit)**
> A mobile user profile screen for a social fitness app, shown in edit mode. A circular avatar at the top with a small camera icon overlay for changing the photo, followed by editable fields: Display Name (filled with 'Jordan Lee'), Bio (a multi-line text area filled with 'Marathon training, one mile at a time.'), and a Location field (filled with 'Austin, TX'). Below the fields, a row of toggle switches for 'Show workout history' and 'Show followers list.' A full-width 'Save Changes' button pinned at the bottom, shown in a disabled/grayed-out state until a field is modified. No delete-account option on this screen.

**Pricing / paywall**
> A mobile pricing screen for a language-learning app, showing three subscription tiers side by side as cards: Basic ($4.99/mo), Plus ($9.99/mo), and Pro ($19.99/mo). The Plus card is visually highlighted as the recommended tier, with a 'Most Popular' ribbon badge and a slightly larger card size than Basic and Pro. Each card lists three feature bullets and a 'Choose Plan' button. Show the Plus card's 'Choose Plan' button in a loading-spinner state to represent an in-progress purchase when tapped. No free-trial banner.

**Settings**
> A mobile account settings screen organized into labeled sections: 'Account' (Name, Email, Password rows, each with a chevron indicating it opens a sub-screen), 'Notifications' (toggles for Push Notifications, Email Digest, and SMS Alerts, with Push Notifications shown on and Email Digest and SMS Alerts shown off), and 'Preferences' (a Language row and a Dark Mode toggle shown off). Each row has a leading icon. At the bottom, outside the sectioned list, a destructive-styled 'Log Out' button and, below it, smaller red text reading 'Delete Account.' No profile photo or avatar upload control on this screen.

**Empty state**
> A mobile screen showing the empty state for a project management app's task list, reached when a user has no tasks yet. Centered content: an illustration of an empty clipboard with a faded checkmark behind it, a heading reading 'No tasks yet,' a subheading reading 'Create your first task to get started,' and a primary button reading '+ New Task.' The top app bar still shows the screen title 'My Tasks' and a disabled filter icon, since the list is empty. A bottom tab bar shows four tabs — Tasks, Projects, Calendar, Profile — with 'Tasks' shown in its active state and the other three inactive. No task list content should be visible.

**Onboarding**
> A mobile onboarding screen, the second of three in a swipeable sequence, for a habit-tracking app. A dot-style page indicator at the top showing the second dot active and the other two dots inactive. Below it, a centered illustration of a seven-day calendar row with six days checked off, a bold heading reading 'Track streaks that stick,' and a supporting sentence reading 'Small daily wins compound. Log a habit each day and watch your streak grow.' At the bottom, a full-width 'Next' button and a smaller 'Skip' text link above it. No back button — this step assumes forward-only navigation within onboarding.

## Device-specific phrasing

The `deviceType` parameter (`MOBILE | DESKTOP | TABLET | AGNOSTIC`) sets the canvas, but the prompt's own wording should match it:

- **`MOBILE`** — describe single-column, vertically stacked layouts; bottom-pinned primary actions and bottom tab bars (not top nav); thumb-reachable controls; swipe/scroll behavior where relevant. Avoid describing multi-column layouts — they don't fit a phone viewport.
- **`DESKTOP`** — describe multi-column layouts, persistent sidebars, data tables with many columns, hover states, and top-anchored navigation bars. It's reasonable to ask for denser information per screen than on mobile.
- **`TABLET`** — a middle ground: describe layouts that can use a two-column split (e.g. a master list plus a detail pane side by side) but keep individual touch targets sized for touch, not mouse precision.
- **`AGNOSTIC`** — describe the content and components without committing to a specific column count or fixed navigation chrome placement; useful for early exploration before a device target is chosen, or for content that's genuinely responsive. Don't use `AGNOSTIC` as a default out of laziness — pick a real device type whenever the project has one, per the pre-flight guidance in the main skill.

## What belongs in the design system, not the prompt

Leave these out of the prompt and let the project's design system (`designSystem: "assets/{id}"`) govern them instead:

- Exact hex color values (e.g. `#1A73E8`)
- Specific font families (e.g. "use Inter")
- Corner radius values (e.g. "8px rounded corners")
- Spacing/sizing tokens (e.g. "16px padding")

Repeating these in every prompt is redundant when a design system is attached, and worse, a prompt-level color or font mentioned once can drift from what's specified in later screens' prompts, undermining the whole point of having a shared design system.

**When it's still justified to override in the prompt:** a genuine one-off exception for this specific screen — e.g. "this error screen should use a red-tinted background to signal severity, overriding the usual neutral background," or "this promotional banner screen should use bold, saturated colors distinct from the rest of the product." State clearly that it's an intentional exception, so it doesn't get mistaken for the new baseline in a future prompt.

## Phrasing for consistency across a set of screens

When generating several screens for the same project:

- Reuse the same descriptive vocabulary for shared components across prompts — if one prompt calls it a "top app bar," don't call it a "header bar" in the next; inconsistent naming invites inconsistent output even under the same design system.
- Explicitly reference shared chrome that should look identical across screens, e.g. "the same bottom tab bar used on the Home screen, with 'Settings' as the active tab" — this anchors the new screen to what already exists instead of letting Stitch reinterpret the tab bar from scratch.
- State the device type's implications consistently (see above) in every prompt in the set, not just the first — no prompt should be the only one that "forgets" to describe mobile-appropriate layout, since that's the screen most likely to end up visually inconsistent with the rest.
- Keep the same `designSystem` and `deviceType` arguments across every call in the set — this is an API-level consistency mechanism, not just a prompting one, and it does more for visual consistency than prompt wording alone.
