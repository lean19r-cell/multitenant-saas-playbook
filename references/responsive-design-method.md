# Responsive design: process and the specific pattern that held up

## Mockup before code, for anything visual and non-trivial

Before touching real components for a layout/visual change bigger than a one-line tweak: build a throwaway, self-contained HTML mockup using the *actual* design tokens (real color values, real font stack, real spacing scale — copy them out of the real config, don't approximate) and real-shaped example data, publish it somewhere the user can look at it, and get a yes before writing the real diff. This is cheap insurance against building the wrong thing in real components, where "just tweak it" costs a rebuild-and-reverify cycle instead of a five-minute edit to a static file. It also surfaces genuine open questions (exact breakpoint, exact max-height of a scrollable region) as concrete choices the user can react to, rather than assumptions buried in a diff they're unlikely to scrutinize as closely as a picture.

Skip the mockup step for things that are really bugfixes wearing a visual costume (an invisible chart bar, a misaligned badge) — the mockup ritual is for *new* design decisions, not for restoring something to how it was clearly supposed to look.

## Retrofitting responsiveness onto a desktop-only app

A common starting state: an app built and shipped against one implicit target width, with zero responsive breakpoints anywhere, that a user only discovers once they open it on a phone or a half-width browser window. The fix is rarely "add one breakpoint" — it's usually a small number of *repeating shapes* across many screens:

1. **A fixed-width side rail (main nav) that eats a fixed number of pixels regardless of viewport** — on a real phone width this alone can be a quarter of the screen. Fix: hide it below your chosen breakpoint, and if the app is used heavily on mobile (a point-of-sale app staff carry around, say), replace it with a bottom tab bar rather than a hidden hamburger drawer — a POS in particular gets its nav touched constantly through a shift, and a drawer adds a tap to every single navigation.
2. **A two-column "list + detail panel" layout with a fixed-pixel-width panel** (`w-80`, `w-[420px]`, whatever) that was never given a narrow-viewport fallback. Fix: stack to one column below the breakpoint (panel moves below the list, both full width), keep the fixed width only from the breakpoint up. This exact shape tends to repeat verbatim across every screen that follows a "select an item, see its details on the side" pattern — find the first one, then grep for the same wrapper classes on every other screen, since they were very likely built from the same starting template.
3. **Any table with a fixed or wide implicit width** — wrap it in its own horizontally-scrolling container (`overflow-x-auto`) rather than trying to make the table itself responsive; a table genuinely needs all its columns, scrolling sideways is the honest way to keep them all reachable on a narrow screen.
4. **A filter/nav sidebar *inside* a page** (a "13 report types" list, say) that's a second fixed-width column on top of the app's own main nav — easy to miss because it doesn't look like "the sidebar," it looks like page content. On narrow viewports this stacks with the main nav to eat even more width than either alone. A native `<select>` covering the same options, shown only below the breakpoint, is a fine, low-effort substitute — it doesn't need to look impressive, it needs to work.

## When the app already has an accordion/list layout: the bugs are small, specific and repeated

A second vertical (property management) never got the pass the first one (POS) received. It had no sidebar and no fixed-width side panels — a list with inline expandable detail is mobile-friendly by construction — so a rewrite would have been the wrong response. Investigating in parallel and then checking each screen at 375px in a real browser found a short list of repeating shapes, fixed across ~37 files:

1. **Fixed page padding with no `sm:` variant** (`p-8` on every screen and every `error.tsx`) → `p-4 sm:p-8`. Grep for it; it appears on every page built from the same template.
2. **Rows built as `flex justify-between` with no `flex-wrap`** — a long name plus action buttons pushes the last button off-screen (confirmed live: the "Eliminar" button of a property row was unreachable). Add `flex-wrap`, and give the long text its own full-width line on mobile (`w-full sm:flex-1`).
3. **`min-w-0 flex-1` on that text is not the safe fix.** The first attempt let a long user name wrap onto a second line that rendered *behind* the neighboring `<select>`. `w-full` on mobile and `sm:flex-1` from 640px up fixed it. Look at wrapped text next to a control, not only at whether the row fits.
4. **A page ported from a sibling component doesn't inherit the sibling's mobile fix.** The property-management reports screen is a separate client component from the POS one, so the mobile `<select>` replacing the fixed `w-64` report nav had to be ported by hand. When one screen was fixed, list every other component that copied its layout.
5. **A fixed pixel height on a widget** (a 500px satellite map on a 375px-wide phone) → smaller fixed height below `sm`, original from `sm` up.
6. **Form rows with several inputs plus a button on one line** (the platform admin's "add user" form: four inputs + button) → column on mobile, row on desktop. A filter `<select>` without `w-full` can force horizontal scroll of the *whole page*.

Deliberately left alone, and worth saying so in the PR: tables that already had `overflow-x-auto`, components that already used `min-w-0` + `truncate`, grids that were already `grid-cols-1` mobile-first. Not verified without a real touch device: drawing a polygon with a map-drawing library on a touchscreen.

**Verify before and after at 375px and at desktop width**, on the screens that changed. Don't run a production build in the directory a dev server is using while you do (SKILL.md, "Verify by seeing it").

## Test at the breakpoint boundary, not just the extremes

"Works on my phone" (very narrow) and "works on my monitor" (very wide) are the two easiest states to eyeball and the least likely to be where the actual bug lives. The interesting failures cluster right around the chosen breakpoint — a half-maximized browser window, a small laptop screen, a tablet in portrait. Explicitly resize to a handful of widths spanning the breakpoint (not just below-vs-above) and check both that the narrow layout doesn't look cramped right before the switch and that the wide layout doesn't look starved right after it.

When something a user reports as "broken" is actually the deliberately-narrow-viewport version of correct, working-as-designed behavior, that's worth checking for directly — reproduce their exact scenario (same viewport width, ideally the same device class) before assuming a fresh bug rather than a scope misunderstanding.

## "Sticky"/floating panels are a real, separate design decision — don't assume the user wants it just because it's technically nicer

A panel that stays fixed on screen while its sibling content scrolls (`position: sticky`) is a genuinely different interaction than a panel that scrolls away normally with the page, and reasonable people can prefer either depending on how tall the sibling content usually is, how often they need to glance at the fixed panel mid-scroll, and just personal taste. It's extremely common for a user to ask for "sticky" by name (because a competitor's app does it, or because it sounds objectively better), get it built exactly as asked, and then — only once they're actually scrolling through it during real use — decide they don't like it and want the panel to scroll normally after all. That's not a sign the first implementation was wrong; it's the entire reason "try it for real before deciding" is worth doing at all. Implement the reversal exactly as cleanly as the original ask, without treating it as a mistake to explain away.
