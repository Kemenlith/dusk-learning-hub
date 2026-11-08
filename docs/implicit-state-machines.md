# Implicit state machines

This repository is a static site. Each page is one HTML file with CSS and a script at the bottom. There is no shared store, reducer, worker, or API handler.

`index.html` and `index-v9.html` are byte-identical. That pair is the live page. `index-v3.html` through `index-v8.html` are older snapshots still served from the repo root.

**Count: 8 implicit machines.**

Four of them run on the live page. Four exist only in older snapshots.

The phases below are inferred from the source. They are not names the code declares.

## Ranked list

1. Stake price load (`index-v6.html`, `index-v7.html`). The page can sit on "loading..." forever, and the diagram, table, and card can show different prices for the same stake.
2. Scroll reveal (`index-v5.html`, `index-v6.html`, `index-v7.html`). Section text stays invisible until a one-way CSS class is added. A script error leaves the page blank.
3. Scroll-spy section highlight (every HTML file, including the live page). The current section is a CSS class on many links. The highlight can stick on the wrong section when the scroll band is empty.
4. Node-setup panels chosen by paired CSS classes (`index-v4.html`). The selected tab and the visible panel are separate flags. A bad mode string clears both and shows nothing.
5. In-page search filter (`index-v3.html`, `index-v4.html`). A query hides sections by writing `style.display` on each block. A miss hides the whole article and can desync the nav highlight.
6. Mobile navigation drawer (every HTML file, including the live page). Open and closed are two independent CSS classes. Snapshots v5–v7 can open the drawer and cannot close it from the menu button.
7. Search modal (`index-v5.html` through the live page). Open and closed are a CSS class. Result content is recomputed from the input string. The footer describes an Enter-to-navigate step the script does not track.
8. Copy-button feedback (every HTML file, including the live page). Success is button text plus a CSS class, cleared by a timer. A second click lets the first timer wipe the newer success. Clipboard failure has no phase.

## 1. Stake price load

**Files and symbols.** `index-v6.html` and `index-v7.html`, IIFE `fetchPrices`. DOM ids: `svg-eth-price`, `svg-dusk-price`, `table-eth-price`, `table-dusk-price`, `card-stake-text`.

**Implicit representation.** No status field. Phase is whatever text those nodes currently hold. The SVG nodes start as the string `loading...`. The table starts as `~$96,000+` and `~$30`. The card starts as `1,000 DUSK vs 32 ETH.` `fetchPrices` writes a node only when the matching price is truthy.

**Inferred states.**

- Waiting. The request is in flight, or it never started. The SVG still says `loading...`.
- Partial. One asset updated. The other still shows its initial string.
- Full. Both ETH and DUSK numbers are written. The card sentence is rewritten only when both prices exist.
- Failed. The `catch` runs. Every node keeps its initial string.

Waiting and failed look the same on the diagram.

**Inferred transitions.**

- Page load calls `fetchPrices` once.
- HTTP and JSON success with both `ethereum.usd` and `dusk-network.usd` moves to full.
- Success with only one price moves to partial.
- A price of `0` stays on the initial string, because the code treats it as missing (`if (eth)`, `if (dusk)`).
- Network error, non-JSON body, or thrown error moves to failed. The handler is `.catch(() => { /* silent fail — fallback text stays */ })`.
- There is no timeout, abort, or retry.

**Illegal combinations the current types allow.**

- Diagram says `loading...` while the table shows a dollar amount.
- ETH updated and DUSK still `loading...`, or the reverse.
- Card text updated while one SVG node is still `loading...` cannot happen in this handler (the card write requires both). The DOM still allows that mix if anything else edits one node.
- After failure, the table looks like a finished quote and the diagram looks unfinished.

**Why it matters.** A reader comparing stake cost can trust a hardcoded fallback, or watch "loading..." that never resolves. The two surfaces disagree. The comment calls the leftover text a fallback. On the diagram, the leftover text is the loading placeholder.

**Confidence.** High.

## 2. Scroll reveal

**Files and symbols.** `index-v5.html`, `index-v6.html`, `index-v7.html`. CSS `.section` and `.section.revealed`. Script symbol `revealObserver`. Hero is forced with `document.getElementById('hero').classList.add('revealed')`.

**Implicit representation.** A section is hidden when it lacks `revealed`. `.section` sets `opacity: 0`. `.section.revealed` sets `opacity: 1`. The observer adds the class and then calls `unobserve`. Nothing removes the class.

**Inferred states.**

- Hidden. The section has not intersected the reveal root.
- Shown. The class is present. This does not reverse.

Hero is shown on load even before it intersects. Every other `.section` starts hidden.

**Inferred transitions.**

- Load shows hero only.
- A section that intersects (`rootMargin: '0px 0px -80px 0px'`, `threshold: 0.05`) becomes shown.
- Leaving the viewport does not hide it again.
- There is no error or timeout path. If the observer never fires, the section stays hidden.

**Illegal combinations the current types allow.**

- `revealed` present while inline style or another rule keeps opacity at 0.
- A section the user has scrolled to, still without `revealed`, when the threshold or bottom margin is not met.
- Hero shown and every other section hidden for the whole visit, if the observer is never created.

**Why it matters.** Before the script runs, every section, including the hero, is invisible. If the script throws first, the article stays blank. On v5–v7 that is the whole page. The live page removed this (`opacity: 1`, and a comment that sections are visible by default). The snapshots still have it.

**Confidence.** High.

## 3. Scroll-spy section highlight

**Files and symbols.**

- Live page and `index-v5.html` through `index-v8.html`: `IntersectionObserver` callback over `.section[id]`, class `active` on `.nav-link[data-section]`. Options: `rootMargin: '-20% 0px -70% 0px'`, `threshold: 0`.
- `index-v3.html` and `index-v4.html`: same root margin on `.section[id]`, class `active` on `.nav-item`. The match key is `href`, not `data-section`.

The live nav starts with `active` on the Overview link (`data-section="hero"`). v3/v4 start with `active` on `href="#overview"`.

**Implicit representation.** There is no `currentSection` value. The phase is whichever links carry `active`. The live callback sets that class only inside `if (entry.isIntersecting)`. It never clears the highlight when a section leaves the band.

The band is the strip from 20% to 30% of the viewport. That is what the negative root margin leaves.

**Inferred states.**

- Highlight on one section id. After a callback, every observed link is toggled so only links whose `data-section` equals that id stay active. v3/v4 remove `active` from all items, then add it to the href match.
- Highlight stuck on the last id that entered the band.
- Highlight on none. This happens when an intersecting id has no matching link. The live pages currently have one link per section id (`hero` through `resources`). v3/v4 ids also match their hrefs.

**Inferred transitions.**

- Scroll or hash jump brings a section into the band. The highlight moves to that id.
- Two sections in the band during one callback: the last entry in the observer list wins.
- The section leaves the band and nothing else enters. The highlight stays.
- A nav click scrolls. It does not set `active` itself. The observer is the only writer after load.
- v3/v4 also hide sections during search (machine 5). A hidden section can still be the last intersecting id, or can stop intersecting and leave the highlight on a link whose row is `display: none`.

**Illegal combinations the current types allow.**

- Several links `active` at once. The HTML can start that way. The live callback avoids it only when some section is intersecting.
- No link `active` while the user is reading a section.
- `active` on a section that is above or below the band, including the initial Overview highlight before hero enters the strip.
- In v3/v4, `data-section` and `href` can disagree. The observer reads `href` only, so the extra attribute can name a different section with no effect.

**Why it matters.** The sidebar shows the wrong section. At the bottom of a long page the last band exit leaves the previous item lit. Fast scrolling can flicker when two entries are intersecting. On v3/v4 the filter and the highlight share the same links and can mark a hidden item.

**Confidence.** High.

## 4. Node-setup panels chosen by paired classes

**Files and symbols.** `index-v4.html` only. `switchSetup(mode)`, buttons `.setup-tab`, panels `#setup-standard` and `#setup-separated`. CSS: `.setup-panel { display: none }`, `.setup-panel.active { display: block }`, `.setup-tab.active` for the selected button.

**Implicit representation.** `mode` exists only as the function argument (`'standard'` or `'separated'` from an `onclick` string). Nothing stores the mode afterward. The phase is the `active` class on a button plus the `active` class on a panel. The button is found with `.setup-tab[onclick*="' + mode + '"]`.

**Inferred states.**

- Standard. The standard button and `#setup-standard` have `active`. This is the initial markup.
- Separated. The separated button and `#setup-separated` have `active`.
- Blank. No panel has `active`.
- Split. The lit button and the visible panel name different modes, or more than one panel is active.

**Inferred transitions.**

- Click "Standard Setup" or "Separated Key Setup" calls `switchSetup`. It strips `active` from every tab and every panel, then adds it to `#setup-` plus the mode, then to the matching button.
- Any other string still strips every `active` first. `getElementById('setup-' + mode)` is null, and reading `.classList` throws. The button line never runs. The UI stays blank.
- There is no cancel or default recovery.

**Illegal combinations the current types allow.**

- Both panels active, or neither.
- A tab active with the other panel active.
- Button lookup by substring. A future mode whose name contains `standard` can light the wrong button after the panel id was set.

**Why it matters.** The reader follows the wrong setup (one wallet versus a cold owner key) or sees an empty guide. Later snapshots replaced this with a radio group. See "Already explicit".

**Confidence.** High.

## 5. In-page search filter

**Files and symbols.** `index-v3.html` and `index-v4.html`. Input `#search-input`. The `input` listener walks `.section[id]` and `.nav-item`. Ctrl/Cmd+K focuses the input. It does not open a dialog.

**Implicit representation.** The query string is the only input. Each section and each nav item stores its own phase in `element.style.display` (`''` or `'none'`). There is no result list and no "no matches" node.

**Inferred states.**

- Unfiltered. The query is empty. Every section and nav item has `display` cleared.
- Filtered. Some sections match `textContent`, so they stay shown. The rest are `display: none`. A nav item is shown when its target section exists and that section's `display` is not `'none'`.
- None visible. The query matches nothing. Every section is hidden.

**Inferred transitions.**

- Each `input` event recomputes both lists from scratch.
- Clearing the query returns to unfiltered.
- Ctrl/Cmd+K moves focus to the field. It does not change the filter.
- There is no Escape handler and no error path.

**Illegal combinations the current types allow.**

- A section hidden and its nav item shown. The nav rule treats any `display` other than the exact string `'none'` as visible. A section set to `display: ''` with an ancestor hidden would still show the link.
- A section shown and its nav item hidden, if the href does not match the id (`href.slice(1)`).
- All sections `display: none` while a link still has `active` from the scroll-spy.
- `display` values other than `''` and `'none'` on a section, which this listener will overwrite on the next input, but other code can set in between.

**Why it matters.** A query with no hit removes the article with no empty-state message. The nav highlight from machine 3 keeps running on sections this filter has hidden, so the lit item and the visible page can diverge.

**Confidence.** High.

## 6. Mobile navigation drawer

**Files and symbols.**

- Live page: `#sidebar`, `#sidebar-overlay`, click on `#hamburger`, click on `#sidebar-overlay`, click on `.nav-link` inside the sidebar. CSS under `@media (max-width: 768px)`: `#sidebar.open` slides the nav in, `#sidebar-overlay.open` shows the dimmer.
- `index-v5.html`, `index-v6.html`, `index-v7.html`: the hamburger only calls `classList.add('open')` on both nodes.
- `index-v8.html` and the live page: the hamburger `toggle`s both classes.
- `index-v3.html` and `index-v4.html`: inline `onclick` toggles `.sidebar` and `.overlay`. The overlay click and each `.nav-item` click remove `open` from both.

The button label stays `aria-label="Open navigation"` in every phase. Nothing sets `aria-expanded`.

**Implicit representation.** Two independent class flags. No single open boolean. Desktop CSS ignores both flags (the sidebar is the normal column, and the overlay rule is inside the mobile media query).

**Inferred states.**

- Closed. Neither node has `open`. Initial markup.
- Open. Both have `open`. On a narrow viewport the drawer is visible and the page is dimmed.
- Menu only. Sidebar has `open`, overlay does not.
- Dimmer only. Overlay has `open`, sidebar does not.
- Latent. Both have `open` while the viewport is wider than 768px. The classes are set and the mobile rules are not applying.

**Inferred transitions.**

- Live hamburger toggles both flags together.
- Overlay click and nav-link click remove both flags.
- v5–v7 hamburger only adds `open`. Closing requires the overlay or a nav link. Pressing the button again while open does nothing.
- Resize does not add or remove the classes. Crossing 768px only changes whether the classes paint.
- There is no Escape path and no focus trap.

**Illegal combinations the current types allow.**

- Menu only, and dimmer only. Every call site writes both nodes, but the DOM does not couple them. An exception on the second line, or any other script, leaves the split.
- `open` on a desktop viewport, which becomes a real drawer the moment the window shrinks.
- v5–v7: open with no control on the button that opened it. The user must hit the dimmer. If the dimmer failed to receive `open`, the drawer has no close control in that script.

**Why it matters.** A split shows a drawer that does not dismiss on outside click, or a dimmer with no menu. On v5–v7 the menu button cannot close what it opened. The live page toggles, and still keeps the phase in two classes that must be edited together.

**Confidence.** High.

## 7. Search modal

**Files and symbols.** `index-v5.html` through `index-v9.html` and `index.html`. Functions `openSearch`, `closeSearch`, `renderResults`, `highlight`. Data `SEARCH_DATA`. Nodes `#search-overlay`, `#search-input`, `#search-results`. CSS: `#search-overlay { display: none }`, `#search-overlay.open { display: flex }`. The overlay is `role="dialog"` and `aria-modal="true"` in the markup, including while it is closed.

**Implicit representation.** Closed versus open is the presence of class `open`. The query is the input's value. The result phase is not stored. `renderResults` derives it: empty query lists the first 10 entries, a query with hits lists up to 10 anchors, a query with no hits replaces the list with `.search-empty`. `closeSearch` removes `open` and leaves the input and the result DOM as they are. The next `openSearch` clears the input and renders the full list.

**Inferred states.**

- Closed. Overlay has no `open`. Result nodes from the last open may still be in the hidden dialog.
- Open, browsing. Query is empty. Ten catalog rows are shown.
- Open, matches. Query is non-empty and at least one row matched.
- Open, empty. Query is non-empty and the empty message is shown.

The footer text says Enter navigates and Escape closes. Escape is implemented. There is no selected-row index, and no Arrow handler. Enter follows a focused anchor only, because the rows are normal links.

**Inferred transitions.**

- Button `#open-search`, button `#mobile-search`, or Ctrl/Cmd+K calls `openSearch`. If the dialog is already open, that path clears the query and shows the catalog again.
- Input events call `renderResults`.
- Escape calls `closeSearch` even when the dialog is already closed.
- A click on the overlay backdrop, outside `#search-modal`, closes. A click on a result calls `closeSearch` and follows `href`.
- `openSearch` focuses the input in a 50ms timeout. Closing does not move focus back.
- There is no loading or error phase. The catalog is a local array.

**Illegal combinations the current types allow.**

- Class `open` with a result list that does not match the current input, if `renderResults` did not run after a value change.
- Dialog semantics (`aria-modal="true"`) while `display` is `none`.
- Closed overlay whose result DOM still contains the previous query's rows or empty message.
- A footer that tells the reader to navigate with Enter, while the script has no cursor among the rows.

**Why it matters.** Re-opening resets a query the user had typed, because Ctrl+K always calls `openSearch`. Escape while typing in another control also calls `closeSearch`. The dialog stays exposed to assistive tech as a modal when it is hidden. Stale rows are invisible, until something else shows the overlay without rendering.

**Confidence.** High for the open flag and the three result presentations. Medium for how often the stale-row case shows up, because every current writer renders on open and on input.

## 8. Copy-button feedback

**Files and symbols.** Every HTML file.

- Live page and v5–v8: listener on each `.copy-btn`. On success it sets `textContent` to `Copied!`, adds class `copied`, and after 2000ms sets `Copy` and removes `copied`. CSS `.copy-btn.copied` turns the label green.
- v3/v4: `copyCode(btn)` sets the text only, and resets after 1500ms. No `copied` class.

The live mobile Search button is `<button class="copy-btn" id="mobile-search">`. It receives the same listener. `closest('.code-block-wrap')` is null, so the handler returns before the clipboard call.

**Implicit representation.** No phase variable and no timer id. Idle versus success is the label string, and on v5+ also the class. The reset is an anonymous `setTimeout` per click. The `clipboard.writeText` promise has no `catch`.

**Inferred states.**

- Idle. Label `Copy`, and on v5+ no `copied` class.
- Success. Label `Copied!`, and on v5+ class `copied`, until that click's timer fires.
- Failed. The promise rejects. The handler does not change the button, so it stays idle. The rejection is unhandled.

There is no in-flight phase. The label stays `Copy` until the write resolves.

**Inferred transitions.**

- Click with a `code` element in the surrounding `.code-block-wrap` (live) or in the parent (v3/v4) starts a write.
- Resolve enters success and arms a timer back to idle.
- Reject stays idle.
- A second click before the timer arms another timer. The older timer still returns the button to idle on its own deadline.
- v3/v4 `copyCode` does not check for a missing `code` node. That path throws before the write.

**Illegal combinations the current types allow.**

- Label `Copied!` without class `copied`, or the class without that label. The success path sets them on two lines, and the timer clears them on two lines.
- Success cleared early while a newer write has already resolved. The first timer does not know about the second click.
- Two timers both resetting an already-idle button.
- `#mobile-search` sitting in the copy-button set. It never enters success. A later edit that wraps it in `.code-block-wrap` would make the Search control show `Copied!`.

**Why it matters.** A double click flashes "Copied!" and then snaps back when the first timer fires, so a successful copy looks failed. A clipboard denial (insecure context, or permission) leaves the label at `Copy` and an unhandled rejection, with no error label.

**Confidence.** High.

## Already explicit

**Node-setup radio group.** Present in `index-v5.html` through the live page. Inputs share `name="node-tab"`. v5 and v6 have `#tab-standard` and `#tab-separated`. v7 through the live page also have `#tab-usb-cold`. `#tab-standard` starts `checked`. CSS shows one label and one panel with `#tab-…:checked ~ …`. The checked radio is the source of truth. The script does not keep a second copy.

The markup still allows a gap the radio group does not close by itself: if no input is checked, every panel stays `display: none` and no label is selected. Setting `checked` on more than one input before the user clicks is also possible from script. The page does not do that.

**Theme (v3 and v4 only).** `let theme` inside the theme IIFE is the writer. Clicks assign `'dark'` or `'light'`, then set `data-theme` on `<html>` and replace the button SVG from that same string. The DOM is a projection of one variable, so this inventory does not treat it as a scattered machine. It is still a bare string. Any other value would set `data-theme` to that string and take the non-dark icon branch. Light-theme CSS is the `[data-theme="light"]` block. The choice is not saved. Reload returns to the initial `data-theme="dark"` and the sun icon in the button markup.

## What was not searched, and what was left out

**Not in this repo.** There are no TypeScript, JavaScript modules, tests, lockfiles, generated bundles, or server handlers to search. Prism and the Google tag are loaded from CDNs. Their internals were not read.

**Read in full for behavior.** The inline scripts in all eight HTML files, the CSS that those scripts depend on (tabs, search, sidebar, copy, reveal, theme), and the markup those scripts query. `index.html` was compared to `index-v9.html` with `cmp` and matched.

**Not executed.** Transitions are from reading the handlers. The pages were not clicked in a browser, so timing bugs (copy timer, 50ms search focus, observer entry order) are inferred.

**Left out on purpose.**

- Consensus, staking maturity, soft and hard slashing, and shield/unshield steps in the article body. Those are descriptions of the Dusk protocol. This site does not run them.
- Color and category labels, hero badges, and ecosystem card groupings. They do not gate a transition.
- `gtag` and Prism. Third-party script tags.
- The theme string, counted above as a single writer rather than an implicit machine.
- The live radio tabs, counted above as an explicit control.
