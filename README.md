# Stackline Full Stack Assignment

## Overview

This is a sample eCommerce website that includes:
- Product List Page
- Search Results Page
- Product Detail Page

The application contains various bugs including UX issues, design problems, functionality bugs, and potential security vulnerabilities.

## Getting Started

```bash
yarn install
yarn dev
```

## 1) Product detail routing: remove JSON-in-URL and route by SKU

- **Major bug**
  - The product detail page was passing the **entire product object as a JSON query parameter** (`?product=...`) in the URL, which created extremely long/fragile URLs and duplicated data that should be fetched by ID.
  - The URL did **not use a stable identifier** for routing despite `stacklineSku` existing.

- **Bug fix**
  - Replaced the home page link to use an ID-based path:
    - From: `/product?product=<JSON-encoded product>`
    - To: `/product/{stacklineSku}`
  - Introduced a dynamic route at `app/product/[sku]/page.tsx` that reads `sku` from params and fetches from `GET /api/products/[sku]` (via `productService.getById(sku)`).
  - Removed the old `app/product/page.tsx` route that depended on the JSON query parameter.

- **Associated improvements**
  - Added explicit loading state (“Loading…”) while the product is fetched by SKU.
  - Implemented clearer error handling for not-found / fetch failures (“Product not found”).
  - Kept the product detail UI effectively identical while making URLs short, shareable, and durable.

## 2) Product detail UI: fix Features section spacing

- **Major bug**
  - The Features section had **excessive padding above the “Features” heading**.
  - Root cause: stacked vertical padding (`Card` provides `py-6`, and the Features `CardContent` additionally added `pt-6`).

- **Bug fix**
  - In `app/product/[sku]/page.tsx`, changed the Features `CardContent` from `CardContent className="pt-6">` to `CardContent>`, removing the redundant top padding while keeping the base `Card` spacing.

- **Associated improvements**
  - Restored visual consistency across card sections by relying on shared `Card` spacing conventions rather than per-page overrides.
  - Reduced the chance of similar spacing drift elsewhere by keeping spacing behavior centralized in the UI primitive.

## 3) Product filtering UI: correct category/subcategory state and fetching

- **Major bug**
  - **Subcategories were not filtered by selected category**: UI fetched `/api/subcategories` without the selected category, so the API received `category === undefined` and returned *all* subcategories (mixing unrelated ones).
  - **Stale subcategory** when switching categories: changing category did not clear `selectedSubCategory`, so the next products request could combine a *new* category with an *old* subcategory.
  - **Loading could get stuck** on network/server errors: `loading` cleared only on success.
  - **Fragile handling of unexpected API response shapes**: missing `categories`, `subCategories`, or `products` could lead to non-array state and `.map()`/`.length` issues.
  - **Products API accepted invalid `limit`/`offset`**: `parseInt` could yield `NaN`, which would flow into pagination logic/response fields.

- **Bug fix**
  - Updated subcategory fetch to include the selected category:
    - `GET /api/subcategories?category=${encodeURIComponent(selectedCategory)}`
  - Cleared stale subcategory immediately on category change (`setSelectedSubCategory(undefined)`), then fetched the new subcategory list.
  - Added basic error handling so `loading` always clears (even on failures) and the UI falls back safely.
  - Hardened response usage with nullish coalescing defaults:
    - `setCategories(data.categories ?? [])`
    - `setSubCategories(data.subCategories ?? [])`
    - `setProducts(data.products ?? [])`
  - Hardened products API pagination params by parsing with base-10, coercing invalid values to defaults, and enforcing bounds (`limit > 0`, `offset >= 0`).

- **Associated improvements**
  - Better UX correctness: category changes no longer leave an invalid subcategory selected.
  - More robust data flow: UI degrades gracefully under network errors or unexpected response shapes.
  - More predictable API behavior: pagination inputs are validated/sanitized.
