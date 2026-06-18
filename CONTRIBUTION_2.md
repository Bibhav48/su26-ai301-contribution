# Contribution 2: Prevent opening glance when selecting text in link

**Contribution Number:** 2  
**Student:** Bibhav Adhikari  
**Issue:** [zen-browser/desktop #8391](https://github.com/zen-browser/desktop/issues/8391)  
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

### Affected Components
* Mouse input systems and event listener dispatch wrappers governing hyperlink interactions.
* The internal logic managing trigger thresholds for opening Zen's native Glance preview window.

---

## Solution Approach

### Analysis
The bug occurred because Zen was evaluating hyperlink interaction rules on basic click-state changes without calculating structural displacement. When a user clicked down, dragged their cursor to highlight text, and let go of the mouse button, the final release action immediately satisfied the baseline criteria to trigger a Glance window event. 

### Proposed Implementation
To differentiate between a deliberate click (to open a link/Glance window) and an intentional click-and-drag movement (to select underlying text), a spatial delta tracking mechanism was required:
1. Capture and save precise mouse coordinates immediately upon the `mousedown` event instance.
2. Measure the exact pixel displacement delta along the X and Y axes when the corresponding `click` event executes.
3. Establish a standard, resilient pixel safety buffer boundary (`CLICK_DRAG_THRESHOLD_PX = 4`) to accommodate minor mouse jitter during a standard click event.
4. Suppress the Glance window trigger entirely if the cumulative layout displacement exceeds the 4-pixel movement threshold, treating the action as an intentional selection block.

---

## Implementation Notes

### Code Changes
* **Files modified:** 1 core file handling workspace click interactions changed.
* **Impact footprint:** +20 structural lines added, 0 deletions.
* **Key implementation commit:** `d1e8f72` (*gh-8391: Prevent opening glance when selecting text in link*)

---

## Pull Request
**PR Link:** [zen-browser/desktop #14164](https://github.com/zen-browser/desktop/pull/14164)  
**Review Status:** Approved by code owner `mr-cheffy` and validated via automated `Copilot` review sequences.  
**Final Status:** Merged successfully into the core `zen-browser:dev` production branch.