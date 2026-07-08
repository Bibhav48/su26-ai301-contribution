# Contribution 2: Prevent opening glance when selecting text in link

**Contribution Number:** 2  
**Student:** Bibhav Adhikari  
**Issue:** [zen-browser/desktop #8391](https://github.com/zen-browser/desktop/issues/8391)  
**Fork:** [Bibhav48/desktop](https://github.com/Bibhav48/desktop)  
**Status:** Phase IV: Complete (Merged)

---

## Why I Chose This Issue

I chose this issue because I personally faced this exact problem while using Zen Browser on macOS. Since I frequently highlight text within links, having the Glance preview disrupt this action was highly frustrating. The fix was relatively simple: tracking mouse displacement to differentiate between a standard click and a text-selection drag.

---

## Understanding the Issue

### Problem Description

Since Zen Browser is built on Firefox (using the Gecko layout engine), it inherits Firefox's native behavior where holding down the `⌥` (Option) key on macOS (or `Alt` on Linux/Windows) allows a user to click and drag across a hyperlink to select the raw text inside it instead of navigating away. However, executing this interaction in Zen Browser unexpectedly triggered Zen's custom "Glance" preview window layout upon mouse release, conflicting with the text selection and disrupting the user's workflow.

### Expected Behavior

When a user holds the modifier key and highlights text inside a hyperlink, the browser should interpret a sustained cursor movement as a text-selection event, successfully highlighting the targeted characters while safely suppressing the "Glance" layout preview mechanism.

### Current Behavior

When holding the Option (⌥) / Alt key and dragging over a hyperlink to select text, Zen Browser interprets the mouse release as a click event on the hyperlink. This triggers the custom Glance feature, opening a preview overlay window and disrupting or obscuring the selected text.

### Affected Components

*   Mouse input systems and event listener dispatch wrappers governing hyperlink interactions.
*   The internal logic managing trigger thresholds for opening Zen's native Glance preview window.

---

## Reproduction Process

### Environment Setup

*   **Operating System:** macOS Tahoe (26.x) running on arm64 Apple Silicon.
*   **Environment:** Local Zen Browser desktop development environment built from source on macOS (arm64) using standard compilation scripts and build guidelines.

### Steps to Reproduce

1. Open Zen Browser.
2. Navigate to any webpage containing text hyperlinks (e.g., Wikipedia).
3. Hold down the `⌥` (Option) key (on Mac) or `Alt` key (on Windows/Linux).
4. Click and drag the cursor across a hyperlink to select the text inside it.
5. Release the mouse button.

### Reproduction Evidence

*   **Commit showing reproduction:** N/A (Reproduction performed locally on developer builds).
*   **Screenshots/logs:** Under the previous behavior, the event listener registered to capture link clicks did not verify if the mouse had moved significantly during the click action. Thus, drag actions were treated as simple clicks, unexpectedly launching the Glance overlay.
*   **My findings:** Discovered that the mouse listener registered to capture link clicks for Glance previewing did not inspect mouse displacement coordinates between `mousedown` and `mouseup`/`click` events.

---

## Solution Approach

### Analysis

The bug occurred because Zen was evaluating hyperlink interaction rules on basic click-state changes without calculating structural displacement. When a user clicked down, dragged their cursor to highlight text, and let go of the mouse button, the final release action immediately satisfied the baseline criteria to trigger a Glance window event.

### Proposed Solution

To differentiate between a deliberate click (to open a link/Glance window) and an intentional click-and-drag movement (to select underlying text), a spatial delta tracking mechanism was required:
1. Capture and save precise mouse coordinates immediately upon the `mousedown` event instance.
2. Measure the exact pixel displacement delta along the X and Y axes when the corresponding `click` event executes.
3. Establish a standard, resilient pixel safety buffer boundary (`CLICK_DRAG_THRESHOLD_PX = 4`) to accommodate minor mouse jitter during a standard click event.
4. Suppress the Glance window trigger entirely if the cumulative layout displacement exceeds the 4-pixel movement threshold, treating the action as an intentional selection block.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Prevent the Glance preview window from opening when a user selects text within a link using Option/Alt + Drag.

**Match:** In standard web browsers (like Firefox, which Zen is based on), a drag of a certain distance (threshold) is distinguished from a click. I need to implement a similar displacement delta check when handling clicks that trigger the Glance feature.

**Plan:**
1. Hook into the mouse interaction logic where the Glance trigger is evaluated on link click.
2. Store the `clientX` and `clientY` coordinates during the `mousedown` event.
3. During the `click` or `mouseup` event, retrieve the coordinates, compute the Euclidean distance (or coordinate deltas) from the stored `mousedown` coordinates.
4. If the distance exceeds `4` pixels (the threshold), mark the action as a drag/text selection and prevent the Glance window from being triggered.

**Implement:** Committed the changes under commit [`d1e8f72`](https://github.com/zen-browser/desktop/commit/d1e8f72).

**Review:** Checked that the threshold works across different screen resolutions and scaling factors without causing false negatives for standard clicks.

**Evaluate:** Verified manually that Option + dragging correctly highlights text without opening the Glance window, while a normal Option + click still successfully launches Glance.

---

## Testing Strategy

### Unit Tests

*   [x] Verify displacement calculations return correct values for various starting and ending coordinate pairs.
*   [x] Test that coordinate deltas below the `CLICK_DRAG_THRESHOLD_PX` threshold do not suppress the Glance trigger.

### Integration Tests

*   [x] Verify that holding Option and clicking (displacement < 4px) still launches the Glance window.
*   [x] Verify that Option + dragging (displacement >= 4px) successfully highlights link text and prevents the Glance window from spawning.

### Manual Testing

*   Verified manually by compiling the browser locally, navigating to a text-heavy page (e.g. Wikipedia), and repeating text selection drags across links multiple times. Tested both trackpad gestures and external mouse movement.

---

## Implementation Notes

### Week 1 Progress

*   Analyzed the Glance window invocation handler. Identified where the browser captures mouse events on links.
*   Debugged how Firefox's native text selection (`⌥` + drag) interacts with Zen's custom click listeners.

### Week 2 Progress

*   Designed the displacement tracking algorithm.
*   Implemented the coordinates storage on `mousedown` and distance evaluation on `click`/`mouseup`.
*   Tuned the jitter buffer threshold to `4` pixels after testing values between `2` and `8`.

### Week 3 Progress

*   Cleaned up the event listener code to prevent memory leaks from dangling mouse coordinate state.
*   Committed the fix and submitted the PR to `zen-browser/desktop`.

### Code Changes

*   **Files modified:** 1 core file handling workspace click interactions changed.
*   **Key commits:** [`d1e8f72`](https://github.com/zen-browser/desktop/commit/d1e8f72) (*gh-8391: Prevent opening glance when selecting text in link*)
*   **Approach decisions:** Differentiating clicks from drags using a pixel displacement delta is standard and highly reliable across platforms, avoiding reliance on OS-level drag-drop events which can be unpredictable inside web layout frames.

---

## Pull Request

**PR Link:** [zen-browser/desktop #14164](https://github.com/zen-browser/desktop/pull/14164)  

**PR Description:** This pull request prevents the Glance window from opening when a user selects text within a hyperlink by holding the modifier key (Option/Alt) and dragging the mouse. By tracking the mouse displacement between `mousedown` and `mouseup` events, we ignore the Glance trigger if the movement exceeds a 4-pixel threshold, indicating a drag/selection action rather than a click.

**Maintainer Feedback:**
*   Approved by code owner `mr-cheffy`.
*   Validated via automated `Copilot` review sequences.

**Status:** Merged

---

## Learnings & Reflections

### Technical Skills Gained

*   Deep understanding of Firefox/Gecko mouse event propagation and coordinate tracking API.
*   Learned about Zen Browser's custom Glance layout UI and how it intercepts web interactions.

### Challenges Overcome

*   Distinguishing small user hand jitters from intentional text selections. Overcame this by introducing and tuning the pixel threshold to `4px`.

### What I'd Do Differently Next Time

*   Investigate if there is an existing Firefox utility class or threshold constant for drag-distance thresholds to match system defaults more closely.

---

## Resources Used

*   [zen-browser/desktop GitHub Repository](https://github.com/zen-browser/desktop)
*   [MDN Web Docs - MouseEvent clientX & clientY](https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent/clientX)
*   [Firefox Source Docs - Event Handling](https://firefox-source-docs.mozilla.org/dom/event/index.html)