# Treets Conversion Attribution (GTM Template)

A [Google Tag Manager](https://tagmanager.google.com/) tag template that measures your
membership's impact by attributing revenue from every order that uses a voucher code that
can be traced back to a treet.

When a checkout happens on your webshop, the tag sends the voucher code, order value, and
currency to Treets. Treets then attributes the order to the right treet — and, when the
code is unique, to the people involved: who redeemed it, who shared the treet, and which
pass the sharer got it from.

## Requirements

- A Google Tag Manager account with access to the website's container.
- A Treets organization with at least one treet.
- Your **Treets organization ID** (Can be found on [hub](https://hub.treets.com/)).
- Voucher codes that can be applied at checkout.

## Installation

1. In Google Tag Manager, open your container and go to **Templates** (or *Custom Templates*).
2. Click **Gallery** and search for **Treets conversion attribution**, or click
   **New → Import Template** and upload the `template.tpl` file.
3. Click **Add** to make the template available in your container.

## Setup

1. Go to **Tags → New**.
2. Give the tag a name, e.g. `Treets conversion attribution`.
3. Under **Tag Type**, choose **Custom Templates** and pick **Treets conversion attribution**.
4. Fill in the fields (see the table below).
5. Under **Triggering**, choose when the tag should fire — typically the event or page view
   on your order confirmation page, after checkout is complete.
6. Click **Save**, then **Publish** the container.

### Fields

| Field | Required | Description |
| --- | --- | --- |
| **Organization ID** | Yes | Your Treets organization ID, provided by Treets. |
| **Order value** | — | The order total at the end of checkout, in whole currency units. For example `10.00` for 10 EUR, **not** `1000` cents. |
| **Currency** | — | An ISO 4217 currency code, such as `DKK` or `EUR`. |
| **Voucher code** | — | The voucher code applied at checkout. If more than one code is applied, separate them with a single separator of your choice — `+`, `,`, or a space — used between every pair of codes. If the field is empty, the tag sends nothing. |

> **Tip:** You can use GTM variables (e.g. from your data layer) for the order value,
> currency, and voucher code so the tag picks them up automatically on every checkout.

## How it works

1. When your trigger fires, the tag sends a single request to
   `https://api.treets.com/api/intreeapi/gtm/checkout` with the organization ID, voucher
   code, order value, and currency.
2. Treets looks up the voucher code(s) for your organization (see
   [Multiple voucher codes](#multiple-voucher-codes) if more than one was applied):
   - **Unique code** (uploaded via CSV): the order is attributed to the treet, to the
     person who redeemed it, and to the person who shared the treet (including which pass
     they got it from).
   - **Static code:** the order is attributed to the treet only.
   - **Unknown code:** the request is ignored and nothing is recorded.
3. If the order currency differs from your organization's default currency, the value is
   converted before it is recorded.

### Multiple voucher codes

If more than one voucher code is applied at checkout, put them all in the voucher code
field. Use **one** of `+`, `,`, or a space to separate the codes, and use that same
separator between every pair of codes. Treets splits the value on it, trims whitespace
around each code, and processes them one by one until one matches. For example,
`coupon1+coupon2+coupon3` is read as `coupon1`, `coupon2`, and `coupon3`, and
`coupon1,coupon2` is read as `coupon1` and `coupon2`.

> **Note:** Do not mix separators in a single value, e.g. `coupon1+coupon2,coupon3`. Only
> one separator is used for the split, so the other ones remain glued to the codes and
> they will not match. Pick one separator and use it throughout.

Only the **first** code that matches a treet is attributed. For example, Treet 1 has
`coupon1` and Treet 2 has `coupon2`. An order of 20 DKK arrives with
`coupon1+coupon2` — Treet 1 gets the 20 DKK, Treet 2 gets nothing from this order.

Because the first match wins, the order of the codes matters. If a static code comes
before a unique code, only the treet is attributed (the sharer and redeemer are not).

> [!IMPORTANT]
> Individual codes may not contain `+`, `,`, or a space. When you upload codes
> via CSV, any code containing one of these characters is rejected and the upload fails.

### Privacy & permissions

- The tag sends data **only** to `api.treets.com` (specifically
  `https://api.treets.com/api/intreeapi/*`). No other third parties are contacted.
- No request is sent when the voucher code field is empty.
- Debug logging is available when you inspect your container in GTM's preview mode.

## Troubleshooting

- **No conversions appear in Treets** — check that the tag is actually firing on your
  confirmation page (use GTM preview mode), that the voucher code matches one of your
  Treets codes exactly, and that the organization ID is correct.
- **Code has no effect** — codes are case-sensitive and must match the codes you see in
  Treets (or that were uploaded via CSV) exactly.
- **Code was rejected on upload** — codes may not contain `+`, `,`, or a space, because
  those characters are used to separate multiple codes. Choose codes that avoid them.
- **Only part of a stacked code is attributed** — when several codes are sent, only the
  first one that matches a treet is attributed. Make sure the matching code comes first.
- **Stacked codes are not being split** — you mixed separators in the value (e.g.
  `coupon1+coupon2,coupon3`). Only one separator is used for the split, so the rest stay
  glued to the codes. Use a single separator between every pair of codes.
- **Value looks wrong** — send whole currency units (`10.00`), not minor units
  (`1000` cents).

## For developers

This repository contains the GTM community template source (`template.tpl`) and its
metadata. When modifying `template.tpl`:

- The `___INFO___` section **must always include**
  `"categories": ["ATTRIBUTION"]`:

  ```json
  {
    "type": "TAG",
    "id": "cvt_temp_public_id",
    "version": 1,
    "securityGroups": [],
    "displayName": "Treets conversion attribution",
    "categories": ["ATTRIBUTION"],
    ...
  }
  ```

  Removing this line makes the template disappear from the ATTRIBUTION category in the
  GTM Template Gallery.
- Bump `version` in `___INFO___` for every published change.
- The sandboxed code lives in `___SANDBOXED_JS_FOR_WEB_TEMPLATE___`. Only the GTM
  sandboxed JS API is available (`sendPixel`, `logToConsole`, `encodeUriComponent`,
  `makeString`, ...).
- The `send_pixel` permission in `___WEB_PERMISSIONS___` is scoped to
  `https://api.treets.com/api/intreeapi/*`. If you change the endpoint host, update the
  permission accordingly.
- Automated tests are in the `___TESTS___` section (Google's GTM template test format) —
  run them in your template development environment before publishing.
- The backend that handles these requests lives in `intree-api` (`/gtm/checkout`), which
  validates the voucher code(s) against the organization and performs the attribution:
  - **Multiple codes:** the `coupon` query value is split into codes, each part is
    trimmed, and the first code that resolves to a treet is attributed the whole order.
    The delimiters come from `GTMConfig.coupon_delimiters` (default `["+", ",", " "]`,
    env prefix `GTM_`). **Only one separator is used for the split**: the loop re-runs
    `split()` for every configured delimiter present in the value, so the *last* one in
    the configured order wins and any others are left glued to the resulting parts (and
    therefore will not match). Do not mix separators in a single value, and keep the
    template help text in sync with the configured value.
  - **Code upload validation:** `CodeListT` rejects any uploaded code that contains a GTM
    delimiter (`422`, nothing is stored).
  - **Rate limiting:** the endpoint is rate-limited per client IP (the IP is hashed with
    SHA-256 so it is not stored raw) at `rate_limiter.gtm_requests_per_hour` (default 50).
  - **Response hygiene:** the pixel response carries `Cache-Control: no-store` in
    addition to `X-Content-Type-Options: nosniff`, and monetary values are converted to
    cents with `round()` rather than truncation.

## License

This project is licensed under the [Apache License 2.0](LICENSE).
