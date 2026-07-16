---
name: figma-to-liquid-section
description: "Create one new, standalone Shopify Liquid section from separate desktop and mobile Figma node links, a section name, and optional implementation notes. Use only when the user directly invokes $figma-to-liquid-section for a Figma-to-Shopify section conversion; never invoke this skill implicitly."
---

# Figma to Liquid Section

Create one pixel-accurate, merchant-configurable Shopify section without depending on the host theme's snippets, styles, scripts, or components.

## Accept the input contract

Require these inputs:

1. Desktop Figma node link
2. Mobile Figma node link
3. Section name
4. Optional section-specific notes

Require each Figma link to identify a concrete node. If either link lacks a node ID, ask for a node-specific link before continuing.

Treat notes as the steering channel for choices inside the section, including its Shopify context, blocks, content behavior, carousel behavior, and whether to use Alpine, Swiper, or vanilla JavaScript. Let notes override defaults, but never let them override exact-value extraction, the 1000px breakpoint, scoped CSS, standalone output, or direct Figma asset usage.

Slugify the section name to sections/<slug>.liquid. Create only that new section file. Do not edit templates, snippets, blocks, assets, layout files, configuration, or existing sections. If the target file already exists, stop and ask whether the user wants to replace it or choose another name; never overwrite or auto-suffix it.

## Ground in the repository

Read every applicable AGENTS.md before designing the implementation. Confirm that the repository is a Shopify theme and inspect only enough surrounding code to understand available Liquid context, validation commands, and project pins.

Keep the generated section independent from the host theme. Do not copy or call theme snippets, custom elements, utility classes, event contracts, container classes, CSS variables, or JavaScript helpers.

Determine the intended Shopify page context from the notes, section name, and design. Do not assume product context. If choosing between product, collection, cart, or another object would materially change the section and the context remains ambiguous, ask before writing.

## Extract exact Figma data

Parse each URL into its file key and node ID. Retrieve desktop and mobile design context, preferably in parallel, with Shopify/Liquid, HTML, CSS, and JavaScript identified as the target technologies. Retrieve variable definitions for both nodes. Use metadata to locate smaller child nodes when a frame is too large or a property needs a more targeted read.

Use Figma data as the source of truth for:

- frame and content dimensions
- padding, gaps, alignment, and positioning
- typography family, weight, size, line height, and letter spacing
- colors, gradients, opacity, borders, radii, and shadows
- responsive differences and visibility
- image and SVG asset URLs

Use screenshots only to understand composition and to compare the finished render. Never measure a screenshot or estimate a value from it. If an exact value is absent, perform another targeted Figma read; do not guess.

Use every image or SVG URL returned by Figma MCP literally, including localhost or remotely hosted asset URLs. Do not download the asset, inline or redraw the SVG, install an icon library, create a placeholder, or substitute a similar asset. Keep each URL plainly searchable for the later asset-migration pass. Refresh the design context if a temporary Figma URL expires during implementation.

## Choose bindings and settings

Bind semantic commerce content to the native Shopify object for the intended context:

- use product data for product titles, prices, availability, variants, forms, and product media
- use collection data for collection titles, descriptions, images, and products
- use cart data for line items, quantities, discounts, totals, and cart state
- use another native object when the notes clearly identify another Shopify context

Prefer the native object over a Figma mock asset when an image depicts the actual product, collection, or cart item. Reserve Figma asset URLs for decorative or merchant-authored media.

Move merchant-authored content, labels, links, colors, and media into section settings. Use the Figma content as the schema default when Shopify permits a default. Do not hardcode merchant-visible copy or commerce values in the markup.

Follow these schema rules:

- expose every configurable solid color through a color setting with the exact Figma default
- use color_background when a configurable gradient or CSS background value requires it
- prefer type number for every numeric section setting
- use type range only when the notes explicitly request a bounded slider
- always add padding_top_mobile, padding_bottom_mobile, padding_top_desktop, and padding_bottom_desktop as number settings with exact Figma defaults
- use URL settings for merchant-configurable destinations
- use image_picker settings for merchant-configurable decorative media
- render the selected Shopify image when present and otherwise render the literal Figma asset URL as the temporary fallback

Keep internal layout geometry, typography metrics, radii, borders, and component dimensions as exact scoped CSS constants unless they are genuinely merchant-facing controls. Use blocks when the notes request them or when repeated items must be addable, removable, or reorderable. Add block.shopify_attributes to each rendered block root.

Include one valid section schema, an addable preset, and an appropriate section wrapper tag. Keep setting and block IDs unique.

## Build a standalone section

Place all section markup, CSS, schema, and behavior in the new Liquid file. Use native Shopify Liquid objects, tags, filters, forms, and storefront endpoints as needed, but use no host-theme implementation dependency.

Use a unique, section-derived class prefix. Include section.id in DOM IDs and JavaScript selectors. Scope all CSS beneath #shopify-section-{{ section.id }} so multiple instances cannot leak styles into each other.

Define a local centered container inside the section. Derive its maximum content width and mobile/desktop gutters from the exact Figma frames; do not rely on a theme container or assume that the frame width itself is the content width.

Write mobile CSS as the base cascade. Append desktop overrides under exactly @media (min-width: 1000px). Treat 999px and below as mobile. Do not add another layout breakpoint unless the Figma data or notes explicitly require it. Order general declarations before overrides, keep specificity predictable, and avoid !important.

Reference the four padding settings through section-scoped CSS custom properties. Keep all CSS inline in the section. Do not add a stylesheet link, modify a theme stylesheet, or rely on external Swiper CSS.

Preserve semantic HTML, keyboard operation, focus visibility, labels, control names, image dimensions, and meaningful alternative text. Give decorative images empty alternative text.

## Add interaction only when needed

Use no JavaScript for static sections. Default to Alpine for stateful UI and Swiper for carousels. Use vanilla JavaScript instead when the notes request it. Do not load Alpine when vanilla JavaScript is requested, and do not load Swiper when there is no carousel.

Honor exact library versions pinned by the repository instructions. If none are documented, inspect existing generated sections for a consistent exact pin. If no project pin exists, verify the current stable release from the official project source and pin that exact version; never use latest, a major-version range, or another floating CDN URL.

Load each required library from the section itself with an idempotent, version-keyed window-level promise or equivalent guard. Ensure multiple sections using the same dependency insert and execute it only once. Keep every Swiper style required by the chosen configuration inside the section's scoped style block.

Initialize behavior against the current section root only. Prevent duplicate initialization, clean up listeners and Swiper instances, and support Shopify theme-editor section load and unload events.

## Validate and report

Run the repository's documented Shopify theme validation, normally shopify theme check --fail-level error. Treat RemoteAsset warnings caused solely by approved temporary Figma URLs as expected, but introduce no other new warnings or errors. Run git diff --check and verify that only the intended new section changed.

When a Shopify preview is available, render and compare the section at the exact mobile and desktop Figma frame widths, then verify the cascade around 999px and 1000px. Use screenshots for comparison, not measurement. Test multiple instances and theme-editor reloads for interactive sections.

If preview access is unavailable, complete the exact-value and static validation passes and explicitly report that rendered visual comparison remains unverified. Never claim a rendered comparison that did not occur.

Report the created section path, its Shopify data context, any libraries used, validation results, expected Figma RemoteAsset warnings, and any visual-validation gap.
