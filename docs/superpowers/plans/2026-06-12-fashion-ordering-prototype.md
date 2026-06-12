# Fashion Ordering Prototype Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a polished, mobile-first urban womenswear ordering prototype covering discovery, SKU allocation, cart validation, checkout, and simulated order submission.

**Architecture:** Use a dependency-free single-page application with semantic HTML, modular CSS, and ES modules. Keep catalog data, pure ordering calculations, persisted application state, reusable render components, and page orchestration separate so business behavior can be tested with Node's built-in test runner.

**Tech Stack:** HTML5, CSS3, JavaScript ES modules, Node.js `node:test`, browser `localStorage`, generated local fashion imagery.

---

## File Map

- `index.html`: application shell, viewport metadata, font links, and module entry point.
- `src/styles/tokens.css`: color, type, spacing, radius, and shadow variables.
- `src/styles/base.css`: reset, typography, mobile shell, utilities, and animation.
- `src/styles/components.css`: cards, filters, matrix, cart rows, checkout, and navigation.
- `src/data/catalog.js`: products, categories, store, address, and delivery fixtures.
- `src/domain/order.js`: pure tier-price, quantity, subtotal, and validation functions.
- `src/state/store.js`: application state transitions and local draft persistence.
- `src/ui/icons.js`: small inline SVG icon factory.
- `src/ui/components.js`: reusable HTML render functions.
- `src/app.js`: routing, screen rendering, event delegation, and submit simulation.
- `assets/products/*.webp`: local generated womenswear images.
- `tests/order.test.js`: order calculation and validation tests.
- `tests/store.test.js`: cart state and persistence tests.
- `tests/smoke.test.js`: static project and accessibility smoke checks.
- `scripts/serve.mjs`: local static development server.
- `package.json`: test and preview commands, with no runtime dependencies.

### Task 1: Establish the Tested Application Skeleton

**Files:**
- Create: `package.json`
- Create: `index.html`
- Create: `scripts/serve.mjs`
- Create: `src/app.js`
- Create: `tests/smoke.test.js`

- [ ] **Step 1: Write the failing shell smoke test**

```js
// tests/smoke.test.js
import test from "node:test";
import assert from "node:assert/strict";
import { readFile } from "node:fs/promises";

test("application shell exposes the mobile ordering root", async () => {
  const html = await readFile(new URL("../index.html", import.meta.url), "utf8");
  assert.match(html, /<meta name="viewport"/);
  assert.match(html, /id="app"/);
  assert.match(html, /src="\.\/src\/app\.js"/);
});
```

- [ ] **Step 2: Run the test and verify RED**

Run: `node --test tests/smoke.test.js`

Expected: FAIL with `ENOENT` for `index.html`.

- [ ] **Step 3: Add the package scripts and application shell**

```json
{
  "name": "atelier-ordering-prototype",
  "private": true,
  "type": "module",
  "scripts": {
    "test": "node --test",
    "serve": "node scripts/serve.mjs"
  }
}
```

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="theme-color" content="#f6f1ea">
    <title>序章订货</title>
    <link rel="stylesheet" href="./src/styles/tokens.css">
    <link rel="stylesheet" href="./src/styles/base.css">
    <link rel="stylesheet" href="./src/styles/components.css">
  </head>
  <body>
    <main id="app" aria-live="polite"></main>
    <script type="module" src="./src/app.js"></script>
  </body>
</html>
```

Implement `scripts/serve.mjs` with `node:http`, mapping `/` to `index.html`, rejecting `..` traversal, and serving HTML, CSS, JS, WebP, and SVG MIME types. Add a temporary `src/app.js` that renders `<p>序章订货</p>`.

- [ ] **Step 4: Run the smoke test and verify GREEN**

Run: `npm test`

Expected: 1 test passes.

- [ ] **Step 5: Commit**

```powershell
git add package.json index.html scripts/serve.mjs src/app.js tests/smoke.test.js
git commit -m "chore: scaffold ordering prototype"
```

### Task 2: Implement Pricing and Order Validation

**Files:**
- Create: `src/domain/order.js`
- Create: `tests/order.test.js`

- [ ] **Step 1: Write failing tests for tier prices and allocation totals**

```js
// tests/order.test.js
import test from "node:test";
import assert from "node:assert/strict";
import {
  getUnitPrice,
  countAllocation,
  calculateLine,
  validateLine
} from "../src/domain/order.js";

const tiers = [
  { min: 12, price: 169 },
  { min: 30, price: 155 },
  { min: 60, price: 142 }
];

test("selects the highest reached price tier", () => {
  assert.equal(getUnitPrice(12, tiers), 169);
  assert.equal(getUnitPrice(36, tiers), 155);
  assert.equal(getUnitPrice(60, tiers), 142);
});

test("counts nested color and size allocations", () => {
  assert.equal(countAllocation({ 黑色: { S: 2, M: 4 }, 燕麦色: { M: 3 } }), 9);
});

test("calculates quantity, unit price, and subtotal", () => {
  assert.deepEqual(calculateLine({ allocations: { 黑色: { S: 15, M: 15 } }, tiers }), {
    quantity: 30,
    unitPrice: 155,
    subtotal: 4650
  });
});

test("reports minimum and stock violations", () => {
  const result = validateLine({
    allocations: { 黑色: { S: 8, M: 6 } },
    minimum: 12,
    stock: { 黑色: { S: 7, M: 12 } }
  });
  assert.deepEqual(result, [{ type: "stock", color: "黑色", size: "S", excess: 1 }]);
});
```

- [ ] **Step 2: Run the tests and verify RED**

Run: `node --test tests/order.test.js`

Expected: FAIL because `src/domain/order.js` does not exist.

- [ ] **Step 3: Implement the pure domain functions**

```js
export function countAllocation(allocations = {}) {
  return Object.values(allocations).reduce(
    (total, sizes) => total + Object.values(sizes).reduce((sum, value) => sum + Number(value || 0), 0),
    0
  );
}

export function getUnitPrice(quantity, tiers) {
  return [...tiers]
    .sort((a, b) => a.min - b.min)
    .reduce((price, tier) => quantity >= tier.min ? tier.price : price, tiers[0].price);
}

export function calculateLine({ allocations, tiers }) {
  const quantity = countAllocation(allocations);
  const unitPrice = getUnitPrice(quantity, tiers);
  return { quantity, unitPrice, subtotal: quantity * unitPrice };
}

export function validateLine({ allocations, minimum, stock }) {
  const quantity = countAllocation(allocations);
  const issues = quantity > 0 && quantity < minimum
    ? [{ type: "minimum", missing: minimum - quantity }]
    : [];

  for (const [color, sizes] of Object.entries(allocations)) {
    for (const [size, selected] of Object.entries(sizes)) {
      const excess = Number(selected || 0) - Number(stock[color]?.[size] || 0);
      if (excess > 0) issues.push({ type: "stock", color, size, excess });
    }
  }
  return issues;
}
```

- [ ] **Step 4: Run all tests and verify GREEN**

Run: `npm test`

Expected: 5 tests pass.

- [ ] **Step 5: Commit**

```powershell
git add src/domain/order.js tests/order.test.js
git commit -m "feat: add wholesale pricing rules"
```

### Task 3: Add Catalog Fixtures and Persistent Cart State

**Files:**
- Create: `src/data/catalog.js`
- Create: `src/state/store.js`
- Create: `tests/store.test.js`

- [ ] **Step 1: Write failing state tests**

```js
// tests/store.test.js
import test from "node:test";
import assert from "node:assert/strict";
import { createStore } from "../src/state/store.js";

test("updates a product allocation without mutating other sizes", () => {
  const store = createStore({ storage: null });
  store.setQuantity("coat-01", "酒红", "M", 4);
  store.setQuantity("coat-01", "酒红", "L", 2);
  assert.deepEqual(store.getState().cart["coat-01"], {
    酒红: { M: 4, L: 2 }
  });
});

test("removes zero quantities from the cart", () => {
  const store = createStore({ storage: null });
  store.setQuantity("coat-01", "酒红", "M", 3);
  store.setQuantity("coat-01", "酒红", "M", 0);
  assert.equal(store.getState().cart["coat-01"], undefined);
});

test("restores a saved draft", () => {
  const memory = {
    value: JSON.stringify({ cart: { "coat-01": { 黑色: { S: 2 } } } }),
    getItem() { return this.value; },
    setItem(_, value) { this.value = value; }
  };
  const store = createStore({ storage: memory });
  assert.equal(store.getState().cart["coat-01"].黑色.S, 2);
});
```

- [ ] **Step 2: Run the tests and verify RED**

Run: `node --test tests/store.test.js`

Expected: FAIL because `src/state/store.js` does not exist.

- [ ] **Step 3: Implement state transitions and fixtures**

Implement `createStore({ storage = globalThis.localStorage })` with:

```js
const EMPTY_STATE = {
  route: "home",
  category: "全部",
  query: "",
  selectedProductId: null,
  cart: {},
  checkoutStatus: "idle"
};
```

Expose `getState`, `subscribe`, `navigate`, `setFilter`, `setQuantity`, `removeProduct`, `clearCart`, and `setCheckoutStatus`. Persist only `{ cart }` under `fashion-ordering-draft-v1`, guard malformed JSON, and delete empty color/product objects.

Create 12 catalog records in `src/data/catalog.js`. Every product must contain:

```js
{
  id: "coat-01",
  name: "羊毛混纺收腰大衣",
  styleNo: "AW26-C013",
  category: "外套",
  image: "./assets/products/coat-01.webp",
  wholesalePrice: 169,
  minimum: 12,
  tiers: [{ min: 12, price: 169 }, { min: 30, price: 155 }, { min: 60, price: 142 }],
  colors: ["酒红", "炭黑", "燕麦"],
  sizes: ["S", "M", "L", "XL"],
  stock: {
    酒红: { S: 18, M: 11, L: 6, XL: 0 },
    炭黑: { S: 24, M: 20, L: 14, XL: 8 },
    燕麦: { S: 9, M: 12, L: 7, XL: 3 }
  },
  badges: ["本周新品"],
  fabric: "羊毛 52% · 聚酯纤维 48%",
  fit: "微廓形收腰"
}
```

Also export categories, two stores, addresses, and delivery choices.

- [ ] **Step 4: Run all tests and verify GREEN**

Run: `npm test`

Expected: 8 tests pass.

- [ ] **Step 5: Commit**

```powershell
git add src/data/catalog.js src/state/store.js tests/store.test.js
git commit -m "feat: add catalog and cart draft state"
```

### Task 4: Build the Visual Foundation and Local Product Imagery

**Files:**
- Create: `src/styles/tokens.css`
- Create: `src/styles/base.css`
- Create: `src/styles/components.css`
- Create: `assets/products/*.webp`
- Modify: `tests/smoke.test.js`

- [ ] **Step 1: Extend the smoke test for visual tokens and local assets**

```js
test("visual theme defines the approved palette", async () => {
  const css = await readFile(new URL("../src/styles/tokens.css", import.meta.url), "utf8");
  assert.match(css, /--cream:\s*#f6f1ea/i);
  assert.match(css, /--ink:\s*#191816/i);
  assert.match(css, /--wine:\s*#7a2332/i);
  assert.match(css, /--sage:\s*#647563/i);
});

test("catalog images use local webp assets", async () => {
  const source = await readFile(new URL("../src/data/catalog.js", import.meta.url), "utf8");
  assert.doesNotMatch(source, /https?:\/\//);
  assert.match(source, /assets\/products\/.+\.webp/);
});
```

- [ ] **Step 2: Run the smoke tests and verify RED**

Run: `node --test tests/smoke.test.js`

Expected: FAIL because the style files and images do not exist.

- [ ] **Step 3: Generate and add the fashion image set**

Use the `imagegen` skill to create a cohesive set of 12 vertical editorial product images:

```text
Premium Chinese urban womenswear wholesale lookbook, full-body studio product
photography, warm limestone and linen backdrop, soft directional daylight,
muted autumn palette, sophisticated commuter styling, garment clearly visible,
no text, no logo, no watermark, vertical 4:5 composition.
```

Create garment-specific variants for coats, knitwear, shirts, dresses, and skirts. Save optimized WebP files at the exact catalog paths, approximately 900 × 1125 pixels.

- [ ] **Step 4: Implement the approved theme**

Define the approved palette, editorial serif display stack, refined Chinese body stack, responsive spacing, 390-pixel phone canvas, subtle grain, focus states, reduced-motion fallback, and staggered screen entrance.

Component CSS must include product grids, editorial hero, category chips, bottom navigation, sticky order bar, filter sheet, color tabs, size matrix, validation notices, grouped cart rows, checkout cards, success state, toast, and loading button.

- [ ] **Step 5: Run tests and verify GREEN**

Run: `npm test`

Expected: all tests pass.

- [ ] **Step 6: Commit**

```powershell
git add src/styles assets/products tests/smoke.test.js
git commit -m "feat: establish editorial ordering theme"
```

### Task 5: Render Home, Catalog, and Product Allocation Screens

**Files:**
- Create: `src/ui/icons.js`
- Create: `src/ui/components.js`
- Modify: `src/app.js`
- Modify: `tests/smoke.test.js`

- [ ] **Step 1: Add failing static accessibility checks**

```js
test("render components expose product and quantity labels", async () => {
  const source = await readFile(new URL("../src/ui/components.js", import.meta.url), "utf8");
  assert.match(source, /aria-label=.*商品/);
  assert.match(source, /aria-label=.*数量/);
  assert.match(source, /aria-current/);
});
```

- [ ] **Step 2: Run the smoke test and verify RED**

Run: `node --test tests/smoke.test.js`

Expected: FAIL because `src/ui/components.js` does not exist.

- [ ] **Step 3: Build reusable render functions**

Export:

```js
renderHeader({ title, back, action })
renderBottomNav(active, cartCount)
renderProductCard(product, { compact })
renderQuantityControl({ productId, color, size, value, stock })
renderAllocationMatrix(product, allocations)
renderStickyOrderBar(summary)
renderNotice(issue)
```

Use buttons for all actions, include product image alt text, include `aria-current="page"` on active navigation, and disable quantity controls where stock is zero.

- [ ] **Step 4: Implement screen orchestration**

`src/app.js` must render:

- `home`: editorial Autumn Commute hero, categories, new arrivals, best sellers, and replenishment banner.
- `catalog`: query input, category chips, sort control, filter action, and two-column cards.
- `product`: gallery, details, color tabs, tier ladder, allocation matrix, next-tier hint, and sticky summary.

Use one document-level click and input handler based on `data-action`. Route changes update state and call `window.scrollTo({ top: 0 })`. “快速订货” opens the product page directly at the matrix section.

- [ ] **Step 5: Run tests and inspect locally**

Run: `npm test`

Run: `npm run serve`

Expected: tests pass; `/`, catalog, and product allocation screens render without console errors.

- [ ] **Step 6: Commit**

```powershell
git add src/ui src/app.js tests/smoke.test.js
git commit -m "feat: build product discovery and allocation"
```

### Task 6: Complete Cart, Checkout, and Submission

**Files:**
- Modify: `src/app.js`
- Modify: `src/ui/components.js`
- Modify: `src/domain/order.js`
- Modify: `tests/order.test.js`

- [ ] **Step 1: Write failing cart-summary tests**

```js
import { calculateCart } from "../src/domain/order.js";

test("cart excludes invalid lines from the payable total", () => {
  const products = [{
    id: "valid",
    minimum: 12,
    tiers,
    stock: { 黑色: { S: 20 } }
  }, {
    id: "invalid",
    minimum: 12,
    tiers,
    stock: { 黑色: { S: 5 } }
  }];
  const cart = {
    valid: { 黑色: { S: 12 } },
    invalid: { 黑色: { S: 8 } }
  };
  assert.deepEqual(calculateCart(cart, products), {
    styles: 2,
    quantity: 20,
    payableQuantity: 12,
    subtotal: 2028,
    hasIssues: true
  });
});
```

- [ ] **Step 2: Run the test and verify RED**

Run: `node --test tests/order.test.js`

Expected: FAIL because `calculateCart` is not exported.

- [ ] **Step 3: Implement cart aggregation**

Add `calculateCart(cart, products)` that calculates each line with `calculateLine`, uses `validateLine`, counts all visible styles and quantities, excludes invalid lines from payable quantity/subtotal, and exposes `hasIssues`.

- [ ] **Step 4: Render cart and checkout**

Add:

- Grouped cart cards with inline allocations and issue notices.
- Empty cart state with return-to-catalog action.
- Disabled checkout while any line has issues.
- Store, address, delivery, note, amount summary, style count, and piece count.
- A 700-millisecond simulated submit state.
- Success screen with generated order number formatted `ORD-YYYYMMDD-NNNN`.
- Failure simulation via a secondary demo control that retains the cart and shows retry.
- Successful submission clears the persisted cart only after the success state is set.

- [ ] **Step 5: Run all tests and manually exercise the full flow**

Run: `npm test`

Run: `npm run serve`

Expected: tests pass; a user can allocate 12+ units, add to cart, check out, submit, and see the success summary.

- [ ] **Step 6: Commit**

```powershell
git add src/app.js src/ui/components.js src/domain/order.js tests/order.test.js
git commit -m "feat: complete ordering checkout flow"
```

### Task 7: Browser Verification and Final Polish

**Files:**
- Modify as required: `src/styles/*.css`
- Modify as required: `src/app.js`
- Modify as required: `src/ui/components.js`
- Modify as required: `tests/*.test.js`

- [ ] **Step 1: Run automated verification**

Run: `npm test`

Expected: all tests pass with zero failures.

- [ ] **Step 2: Start the preview server**

Run: `npm run serve`

Expected: server prints a localhost URL and continues running.

- [ ] **Step 3: Verify in the Browser plugin**

At a 390 × 844 viewport, verify:

1. Home hero and product cards have no horizontal overflow.
2. Category filtering and search change the visible products.
3. Product allocation updates quantity, price tier, subtotal, and next-tier hint.
4. Zero-stock sizes are disabled and low-stock labels are visible.
5. Cart blocks checkout when minimum or stock issues exist.
6. Refresh restores the cart draft.
7. Checkout submits once and displays style/piece counts and an order number.
8. Bottom navigation, back buttons, sticky bars, and focus states remain usable.

- [ ] **Step 4: Capture and inspect key screenshots**

Capture home, product allocation, cart, and checkout screens. Check typography, image crops, sticky-bar overlap, spacing rhythm, warning contrast, and touch target sizes. Apply focused fixes and repeat the affected screenshot.

- [ ] **Step 5: Run final verification**

Run: `npm test`

Run: `git diff --check`

Expected: all tests pass and `git diff --check` emits no output.

- [ ] **Step 6: Request code review**

Provide the reviewer with the design spec, this implementation plan, and the full commit range. Fix all Critical and Important findings, then repeat Step 5.

- [ ] **Step 7: Commit final polish**

```powershell
git add src tests assets package.json index.html scripts
git commit -m "fix: polish ordering prototype interactions"
```

