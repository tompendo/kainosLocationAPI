# SmartAudit — Pendo Location API Demo

A working demonstration of the Pendo Location API built to replicate the Kainos SmartAudit UI. Built in the context of a Pendo POC to show how `addTransforms` can solve the SPA URL problem across two distinct scenarios — one where the browser URL already contains meaningful context, and one where it does not.

---

## The problem

SmartAudit is a Workday-based audit and Segregation of Duties platform built as a Single Page Application. Its core challenge for Pendo is that multiple check types share identical browser URLs — meaning Pendo cannot distinguish whether a user is looking at an Expenses check type, an HCM check type, or a customer-defined check type. They all look the same.

This problem exists at two levels and requires two slightly different approaches, which this demo makes explicit.

---

## The two flows

### Flow A — Business Cycle Conflicts (URL already contains context)

Kainos-defined check types such as *Expenses - SoD - Business Cycle Conflicts* have their category present in the browser URL:

```
Browser URL: /smartaudit/sod/business-cycle-conflicts
```

Pendo can read this directly. The only missing information is which tab the user is currently on. `addTransforms` only needs to **append the active tab state** on top of what is already there.

```
Pendo URL: /smartaudit/sod/business-cycle-conflicts/sod-conflicts-view/review-required
```

### Flow B — Customer Defined View (URL contains no context)

Customer-defined check types all share the same browser URL regardless of which check type is loaded:

```
Browser URL: https://gmsng.eu.stage.kainossmart.com/web/#/audit/checkDetails/221
```

This URL is identical in structure for every customer-defined check type in the platform. The numeric ID changes per check but gives Pendo zero context — SOD, Privileged Access and Config Change customer views are completely indistinguishable from each other. Pendo cannot distinguish between them at all without intervention. `addTransforms` must do two things on every navigation event:

1. **Inject the check type identifier** — available at component initialisation (page load), not in the URL
2. **Append the active tab state** — same as Flow A

```
Pendo URL: https://gmsng.eu.stage.kainossmart.com/web/#/audit/checkDetails/221/expenses-sod-custom-checks/security-groups-view/review-required
```

---

## How the implementation works

### The pendoState module

A single shared JavaScript object holds all current navigation context:

```javascript
const pendoState = {
  checkType:    null,  // null for Flow A (already in URL). Set at page load for Flow B.
  primaryTab:   null,  // which blue tab bar tab is active
  secondaryTab: null   // which white nested tab is active
};
```

Any navigation event updates the relevant fields and calls `fireTransform()`.

### The transform function

```javascript
pendo.location.addTransforms([{
  attr: 'href',
  action: 'Replace',
  data: function() { return buildPendoUrl(); }
}]);
```

`data` is a **function reference**, not a value. Pendo calls it at evaluation time — not at registration time — so it always reads fresh state from `pendoState`. This means:

- No clearing required between tab changes — the function returns a complete, correct URL from current state every time
- No chains to manage — each call to `addTransforms` fires a new pageLoad event with whatever URL the function returns at that moment
- Each pageLoad event is discrete and self-contained in Pendo's data

### What fires the pageLoad event

Calling `pendo.location.addTransforms()` automatically triggers a pageLoad event in Pendo. No explicit call to `pendo.pageLoad()` is required. This was confirmed in the Pendo debugger — a new pageLoad event appears each time `addTransforms` is called, carrying the URL returned by the `data` function at that moment.

### Flow A vs Flow B — the difference in practice

```javascript
// Flow A — context already in browser URL, only tab state needed
updatePendoState({
  checkType:    null,                    // not injected — already in the URL
  primaryTab:   'security-groups-view',
  secondaryTab: 'review-required'
});
fireTransform();
// Pendo sees: /smartaudit/sod/business-cycle-conflicts/security-groups-view/review-required

// Flow B — context NOT in browser URL, must be injected
updatePendoState({
  checkType:    'expenses-sod-custom-checks',  // injected at page load
  primaryTab:   'security-groups-view',
  secondaryTab: 'review-required'
});
fireTransform();
// Pendo sees: https://gmsng.eu.stage.kainossmart.com/web/#/audit/checkDetails/221/expenses-sod-custom-checks/security-groups-view/review-required
```

On every subsequent tab click in Flow B, `checkType` remains in `pendoState` from the initial page load. Only `primaryTab` or `secondaryTab` updates. The injected check type context persists throughout the user's session on that check type without any additional calls.

### Why addTransforms and not setUrl

`pendo.location.setUrl()` disconnects Pendo from the browser URL entirely and requires the application to manually define the URL for every single page — including pages where the browser URL is already perfectly meaningful to Pendo. This would be a significant engineering overhead and a regression for those pages.

`addTransforms` builds on top of the existing browser URL. Pages where the browser URL is already good require zero changes. Only the pages that need context injection (Flow B check types — `#/audit/checkDetails/{id}`) need to write to the shared state module.

---

## URL structure

### Flow A

```
/smartaudit/sod/business-cycle-conflicts/{primaryTab}/{secondaryTab}
```

| Navigation | Pendo URL |
|---|---|
| Page load, default tab | `/smartaudit/sod/business-cycle-conflicts/security-groups-view/review-required` |
| Click SoD Conflicts View | `/smartaudit/sod/business-cycle-conflicts/sod-conflicts-view/review-required` |
| Click Accepted sub-tab | `/smartaudit/sod/business-cycle-conflicts/sod-conflicts-view/accepted` |
| Click Cases View | `/smartaudit/sod/business-cycle-conflicts/cases-view/review-required` |

### Flow B

```
https://gmsng.eu.stage.kainossmart.com/web/#/audit/checkDetails/221/{checkType}/{primaryTab}/{secondaryTab}
```

| Navigation | Pendo URL |
|---|---|
| Page load, default tab | `…/checkDetails/221/expenses-sod-custom-checks/security-groups-view/review-required` |
| Click SoD Conflicts View | `…/checkDetails/221/expenses-sod-custom-checks/sod-conflicts-view/review-required` |
| Click Accepted sub-tab | `…/checkDetails/221/expenses-sod-custom-checks/sod-conflicts-view/accepted` |
| Click Cases View | `…/checkDetails/221/expenses-sod-custom-checks/cases-view/review-required` |

Without the Location API, all four Flow B URLs above would be identical: `#/audit/checkDetails/221`. Pendo would see them all as a single page — and would have no way to distinguish an SOD custom check from a Privileged Access custom check or a Config Change custom check.

---

## Running the demo

No build step, no dependencies, no server required. Open `index.html` directly in a browser or deploy to any static host.

The **Pendo Live debug panel** in the bottom-right shows:
- The raw browser URL — what the SPA actually has in the address bar
- The Pendo URL after `addTransforms` — with the injected segments highlighted
- A live event log of every pageLoad event fired this session, timestamped

Navigate between Flow A and Flow B on the dashboard, then click through the primary and secondary tabs to see the URL updating in real time.

---

## Launching the Visual Designer

To test page and feature tagging against the synthetic URLs, launch the Visual Designer from the browser console:

```javascript
pendo.designerv2.launchDesigner()
```

---

## Key files

```
index.html   — The entire application. Single file, no dependencies.
README.md    — This file.
```

---

## Context

Built by Tom Dowdeswell, Customer Engineer at Pendo, as part of the Kainos SmartAudit POC — April 2026.

Questions: tom.dowdeswell@pendo.io
