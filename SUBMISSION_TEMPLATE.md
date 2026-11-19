# Submission: Kamil Kwiecień

## Time Spent

Total time: **3h 30m** (approximate)

## Ticket Triage

### Tickets I Addressed

List the ticket numbers you worked on, in the order you addressed them:

1. **CFG-142**: Fixed price stuck at $0.00 / wrong total after rapid option changes — removed broken Date.now() on latestRequestRef; kept monotonic request id so only the latest calculatePrice response updates UI. Cleaned dependent add-ons on option change. TechStyle buyers change several options quickly when configuring laptops. The UI sometimes showed a price for an older selection, which undermines trust in checkout.
2. **CFG-148**: Fixed crash when unchecking Include Packaging with Gift Wrap selected — same option-change logic removes add-ons when dependency requiredValue no longer matches (including toggles, not only selects).
3. **CFG-143** (partial): I fixed the window resize listener so it is removed when the component unmounts, which avoids stacking listeners if the widget is mounted again. I’d debounce validation/preview next if QA still sees lag after extended use.
4. **CFG-152**: We have accessibility pressure for the TechStyle evaluation (keyboard / WCAG). I made color swatches and add-on rows usable with Tab and Enter/Space, and improved visible focus so keyboard users aren’t stuck on mouse-only controls.
5. **CFG-149**: After a user changes an option, the price request can take a moment. I made it obvious that a new price is being calculated: the total shows a loading state instead of keeping the old number on screen, and the price area is visually dimmed. The main Add to Cart button stays disabled or shows a short label until the updated total is ready.
6. **CFG-150**: Improved mobile view of this component. In layouts less than 900px I changed buttons flex direction to column.
7. **CFG-156**: No errors in console

### Tickets I Deprioritized

List tickets you intentionally skipped and why:

| Ticket  | Reason                                                                                                                                                                           |
| ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-144 | Product wants to drop Quick Add, but CS and TechStyle still position it as a power-user feature. Removing it without a product decision felt risky right before a prospect demo. |

| CFG-145 | Same conflict as 144. I skipped both. The button still works—hotkey is polish I traded for crash, pricing, and a11y work.|

| CFG-153 | Compare two configurations is a multi week buyer feature, not something we could ship and test in this window.|

| CFG-155 | Dark mode is nice for embed branding but doesn’t fix the reliability issues the demo is judged on.|

| CFG-157 | Unsaved-changes guard is standard UX but needs proper copy, detection rules - too much for the remaining time.|

| CFG-146 | Timestamp offset by timezone is confusing but doesn’t block purchase. I prioritized revenue-impacting bugs first.|

| CFG-151 | End-user error text should come from product/CS, not invented by me under time pressure.|

| CFG-154 | The ticket itself questions whether 50 vs 51 is intentional for the pricing engine. I didn’t change discounts without confirmation.|

### Tickets That Need Clarification

List any tickets where you couldn't proceed due to ambiguity:

| Ticket  | Question                                                                                |
| ------- | --------------------------------------------------------------------------------------- |
| CFG-147 | Inconsistent repro on share URLs - need example configs / encoding rules from Jamie.    |
| CFG-154 | Is quantity 50 supposed to get the 50+ tier, or should we only change the label to 51+? |
| CFG-151 | Can we get approved plain-language strings for network/price/validation errors?         |

### Tickets With a Misleading Priority Label

Our board labels aren't always right. List tickets where you disagreed with the label (e.g. a Low that's actually High, a High that's actually Low, a Medium that's actually Unclear) and explain why. If you didn't disagree with any, say so.

| Ticket | Label on board | Your read | Why |
| CFG-152 | ⚪ Low | 🟠 High | TechStyle compliance and an audit are called out in Context. This is a deal risk, not polish. |

### Tickets Where You Stopped to Ask Instead of Implementing

If a ticket was ambiguous enough that the right move was to escalate / ask PM / gather more context rather than ship a fix, list it here. We value this signal — picking one interpretation blindly and shipping it can cost more than stopping to ask.

| Ticket      | Who would you ask | What you'd ask                                                                                                                                                                                |
| ----------- | ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| CFG-144/145 | PM / CS           | Quick Add is marked for removal (2% usage) while TechStyle wants a keyboard shortcut. Which stakeholder wins for this release? I can’t safely delete or extend the feature without that call. |
| CFG-154     | PM / Finance      | Ticket says 50 items may not get the 50+ tier by design. Changing logic affects quoted order totals for B2B buyers.                                                                           |
| CFG-151     | PM / Finance      | Technical error codes aren’t acceptable for shoppers. Need approved replacement copy and when to show contact support so we don’t ship misleading or legally vague messages.                  |

---

## Technical Write-Up

### Critical Issues Found

Describe the most important bugs you identified:

#### Issue 1: Price never updates — stuck at $0.00

**Ticket(s):** CFG-142

**What was the bug?**

Price fetches could complete out of order or never apply, so the widget showed $0.00, a stale total, or a total that didn’t match the visible options

**How did you find it?**

Traced usePriceCalculation and saw the request guard compared a counter with a timestamp, so valid responses were discarded. Checked handleOptionChange and saw stale selectedAddOns and in-place splice on state.

**How did you fix it?**

Used id only (no overwriting the ref with Date.now()). In handleOptionChange, used setSelectedAddOns(prev => prev.filter(...)) to drop add-ons whose dependency no longer holds when the parent option changes.

**Why this approach?**

I kept the change small on purpose. The widget is already built around async price calls like production embeds, so I fixed the race condition and state bugs instead of rewriting how pricing runs. A bigger debounce on every change would make the total feel sluggish for users who click through options quickly.

---

#### Issue 2: Crash when deselecting "Include Packaging"

**Ticket(s):** CFG-148

**What was the bug?**

If someone turned off Include Packaging but still had Gift Wrap selected, the configurator kept Gift Wrap in the selection even though it only makes sense when packaging is on. When the app tried to use that add-on for price or UI, it hit undefined data and the whole widget crashed with a white screen.

**How did you find it?**

Followed ticket (Premium Material → packaging on → Gift Wrap → packaging off). Mapped to option dependencies in CONTEXT (Gift Wrap requires packaging).

**How did you fix it?**

Same dependency cleanup in handleOptionChange - when any option value changes, remove selected add-ons tied to that option that no longer satisfy requiredValue including toggles, not only select.

**Why this approach?**

I fixed the configuration itself: when an option changes, any add-ons that no longer meet their rules get removed. That way we don’t ship an invalid bundle or a wrong quote. I could have only added null checks in the render path to stop the crash, but the user would still see an impossible combo on screen, and pricing could stay wrong. Cleaning up state matches how the product defines dependencies and it's safer.

---

### Other Changes Made

Brief description of any other modifications:

- Tooling: Targeted ESLint disables on legacy effects where refactor wasn’t worth the time box; kept pre-commit green without rewriting the whole configurator.

---

## Code Quality Notes

### Things I Noticed But Didn't Fix

List any issues you noticed but intentionally left:

| Issue | Why I Left It |
| 800+ line ProductConfigurator | Modular split is a multi-day refactor; risky before a client demo without tests.----------------------------------------------------- |
| Validate + preview on every config change | Likely contributes to CFG-143 sluggishness. Needs debounce + stale response guard |

### Potential Improvements for the Future

If you had more time, what would you improve?

1. If I had more time after the demo, I’d start breaking the big configurator into smaller pieces. Longer term, dependency rules for options and add-ons should live in one obvious place so CS isn’t guessing when someone reports an invalid combo.

---

## Questions for the Team

Questions you would ask in a real scenario:

1. For TechStyle’s evaluation, how far do we need to go on accessibility. Is a full keyboard path from configure to add to cart enough, or is there a formal WCAG checklist we have to meet for the embed?
2. Quick Add is pulled two ways: remove it (144) vs shortcut for enterprise (145). Are any clients still promised that feature?
3. When we show configuration validated does that mean no errors at all, no blocking errors only, or can warnings still appear on a quote a buyer might place?

---

## Assumptions Made

List any assumptions you made to proceed:

1. I assumed the demo is judged on stability and trust: no crashes, the price matches what’s selected, and keyboard use isn’t embarrassingly broken. I treated the dependency rules in Context.md as the source of truth and assumed invalid combos shouldn’t stay in state even if the UI doesn’t crash. I kept async pricing because that’s how the embed behaves in production even though the repo mocks the math.

---

## Self-Assessment

### What went well?

I went after what a buyer would notice first. Writing down what I’d escalate to PM/CS felt closer to how I’d work on a real team than pretending every ticket had one obvious answer.

### What was challenging?

Tickets that contradict each other, pricing/share items where the ticket itself says maybe intended and not having Marcus to ask why the price hook looked the way it did. Pre commit lint also pushed me toward small disables instead of big effect refactors, which is the right trade-off in four hours but not always the prettiest code.

### What would you do differently with more time?

I’d run through a short keyboard and mobile checklist before calling a11y and the color grid done. I’d try debouncing validate preview for the sluggishness ticket. I’d write a message to PM on Quick Add instead of only skipping both tickets silently.

---

## AI Assistance Notes (optional)

If you used AI tools (Copilot, Claude, Cursor, ChatGPT, etc.), we'd love a short note on how. **We don't score the fact that you used AI — we score the quality of your reasoning and verification.**

Useful things to share:

- What kinds of prompts you sent (paraphrase is fine, no need to paste logs)
- Where the AI helped and where you overrode it
- Anything the AI suggested that you rejected — and why
- Anything you verified manually that the AI got wrong

I used Cursor (chat) while working on the task.

1. I was wondering how to skip linting for this project due to short amount of time. Chat pointed me toward targeted eslint-disable comments and making sure Prettier didn’t wipe them, -> Helped me
2. At the end I thought how to fix mobile view in different way and Cursor suggested grid layout for color pickers. After applying it I rejected this and modified buttons above. I still verified layout and price or a11y fixes manually in the browser.

---

## Additional Notes

Anything else you want us to know:

[Your notes]
