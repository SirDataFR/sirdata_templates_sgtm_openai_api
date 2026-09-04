# Optimizing OpenAI (ChatGPT) Ads Conversions API in GTM Server-Side with Enhanced Consent and Data Handling

You can use this tag in the Google Tag Manager Server-Side container to configure the OpenAI Ads Conversions API and manage cookies, data transmission and consent directly on the server with built-in controls.

The tag operates based on GA4 requests and supports custom settings and parameters.

The OpenAI Ads Conversions API serves the same purpose as the OpenAI Pixel, tracking user interactions on your website for ChatGPT Ads campaigns. However, while the Pixel processes requests in the user's browser, the Conversions API operates through a cloud server, enabling various optimizations such as bypassing ad blockers and enriching events with hashed customer data.

This tag works with any container hosting service (Google Cloud, Stape, Sirdata, Taggrs), and is optimized for Sirdata's data enrichment layer, which is compatible with all hosting platforms. This additional layer of data enables several enhancements, including:
- Utilizing TCF consent signals in addition to Consent Mode signals
- Supporting a cookieless user ID when consent is not provided
- Handling consent under GDPR where applicable, and adjusting behavior where GDPR does not apply.

👉 For [Sirdata](https://sgtm.sirdata.io/login) clients, **many additional automatic optimizations are available**, such as integrated third-party ID management, enhanced consent handling, and advanced data enrichment. These features significantly improve OpenAI's match rate, boosting campaign efficiency and tracking precision.

## How it works

1. **Consent gate.** The tag fires only when personalized-ads consent is granted, detected from (in order) the Sirdata `Gtm-Helper-Consent-Personalized-Ads` header (TCF), the Consent Mode `gcs` signal (`ad_storage`), or the `Gtm-Helper-Gdpr-Applies: false` header when no Consent Mode signal is present. Without consent nothing is sent, unless the experimental cookieless mode is enabled.
2. **Event mapping.** GA4 event names are mapped to OpenAI standard events; any other name is sent as a `custom` event with a normalized `custom_event_name`. The `data.type` (`contents`, `customer_action`, `plan_enrollment`, `custom`) is derived automatically from the event type.
3. **Enrichment.** User data is collected from GA4 `user_data`, from Sirdata `Gtm-Helper-*` headers and from the `_gtmeec` first-party cookie (shared with the [Sirdata Meta CAPI tag](https://github.com/SirDataFR/sirdata_templates_sgtm_meta_capi)), then normalized and hashed exactly as specified by OpenAI.
4. **Identifiers.** The OpenAI Click ID (`oppref`) is read from the landing URL, the referrer or the `__oppref` cookie; the Browser ID (`obref`) is read from the `__obref` cookie or generated. Both cookies are (re)written server-side when consent is given and the API accepted the event.
5. **Deduplication.** The GA4 `event_id` is used as the OpenAI event `id`, so events sent by both the OpenAI Pixel and this tag are deduplicated when the same Pixel ID is used.

## GA4 → OpenAI event mapping

| GA4 event | OpenAI event | `data.type` |
| :--- | :--- | :--- |
| `page_view` | `page_viewed` | `contents` |
| `view_item`, `view_item_list`, `select_item` | `contents_viewed` | `contents` |
| `add_to_cart` | `items_added` | `contents` |
| `begin_checkout` | `checkout_started` | `contents` |
| `purchase` | `order_created` | `contents` |
| `generate_lead` | `lead_created` | `customer_action` |
| `sign_up` | `registration_completed` | `customer_action` |
| `schedule` | `appointment_scheduled` | `customer_action` |
| `subscribe` | `subscription_created` | `plan_enrollment` |
| `start_trial` | `trial_started` | `plan_enrollment` |
| `first_open` | `app_installed` | `customer_action` |
| any OpenAI standard event name | that event | as required |
| anything else | `custom` (+ `custom_event_name`) | `custom` |

Amounts (`value`, `items[].price`) are converted to the currency's minor unit as required by the API (42.50 EUR → 4250).

## User data sources

| OpenAI field | Sources (in priority order) | Processing |
| :--- | :--- | :--- |
| `emails_sha256` | `user_data.email`, `user_data.email_address`, `email`, `user_data.sha256_email_address`, `Gtm-Helper-User-Hashed-Email`, `_gtmeec` | trim + lowercase + SHA-256 |
| `phone_numbers_sha256` | `user_data.phone_number`, `phone_number`, `user_data.sha256_phone_number`, `Gtm-Helper-User-Hashed-Phone`, `_gtmeec` | digits only, leading zeros removed, 8-15 digits, SHA-256 |
| `first_names_sha256` / `last_names_sha256` | `user_data.address.first_name` / `last_name` (or `sha256_*`) | lowercase, whitespace and ASCII punctuation removed, SHA-256 |
| `external_ids_sha256` | `external_id`, `user_id`, `Gtm-Helper-Cookieless-Id-Domain-Specific`, `_gtmeec` | trim + SHA-256 (case preserved) |
| `cities`, `regions`, `postal_codes`, `countries` | `user_data.address.*`, then `Gtm-Helper-User-City` / `Gtm-Helper-User-Country` | raw (country uppercased, 2 letters) |
| `ip_address` | `Gtm-Helper-User-Ip`, `ip_override` | raw |
| `user_agent` | `Gtm-Helper-Device-User-Agent`, `user_agent`, `User-Agent` | raw |
| `obref` | `__obref` cookie, `common_cookie.__obref`, `obref`, or generated | raw |

Already hashed values (64 hex characters) are never re-hashed. Values equal to `redacted_for_privacy` are ignored.

## Cookies

| Cookie | Purpose | Lifetime | HttpOnly |
| :--- | :--- | :--- | :--- |
| `__oppref` | OpenAI Click ID, compatible with the OpenAI Pixel | 30 days | no |
| `__obref` | OpenAI Browser ID, compatible with the OpenAI Pixel | 1 year | no |
| `_gtmeec` | Enrichment record (hashed email, hashed phone, external ID) shared with the Sirdata Meta CAPI tag | 1 year | yes |

Cookies are written only when consent is given, the **Generate cookies** option is enabled and OpenAI returned a 2xx response.

## Installation

1. In your GTM Server container, go to **Templates** → **Tag Templates** → **New**, open the three-dot menu, choose **Import** and select `template.tpl`.
2. Create a new tag using the **GDPR Ready OpenAI Ads CAPI by Sirdata** template.
3. Fill in the **Conversions API Key** and **OpenAI Pixel ID** (from the Conversions tab of OpenAI Ads Manager).
4. Trigger the tag on the GA4 events you want to send (e.g. all GA4 events with **Inherit from GA4**).

Use **Validate only** to test the integration without recording conversions.

## Configuration

| Parameter | Description |
| :--- | :--- |
| **Event Name Setup Method** | Inherit from GA4 (default), a fixed OpenAI standard event, or a custom event name. |
| **Conversions API Key** | Mandatory. |
| **OpenAI Pixel ID** | Mandatory. Use the same ID as the browser Pixel for deduplication. |
| **Validate only** | Sends `validate_only: true`; events are validated but not recorded. |
| **Generate cookies** | Writes `__oppref`, `__obref` and `_gtmeec` when consent is given. |
| **Send hashed user data** | Enables email, phone, name, address and `_gtmeec` enrichment. |
| **Send key identifiers** | Enables IP, external ID, user agent, browser ID and coarse location. |
| **Send without consent (experimental)** | Without consent, sends only the Sirdata site-only cookieless ID (hashed), IP and coarse location. No cookie is read or written, no user data is sent. |
| **Item ID key** | GA4 item property used as `contents[].id` (default `item_id`). |
| **Server Event Data Override** | Force `id`, `timestamp_ms`, `oppref`, `source_url`, `action_source`, `opt_out` or `custom_event_name`. |
| **User Data** | Add or override any `user` field; values may be single values or JSON arrays (3 values max per field), hashing is applied automatically. |
| **Event Data** | Add or override any `data` field (`value` in regular unit, `amount` in minor unit, `currency`, `plan_id`, `contents`, custom properties). |

## Useful resources

- [OpenAI Ads Conversions API](https://developers.openai.com/ads/conversions-api)
- [OpenAI Ads supported events](https://developers.openai.com/ads/supported-events)
- [Sirdata GTM Helper layer](https://server-side.docs.sirdata.net/sirdata-server-side/english-1/installation/data-processing/gtm-helper-layer)
- [GDPR Ready Meta/Facebook CAPI by Sirdata](https://github.com/SirDataFR/sirdata_templates_sgtm_meta_capi)

## License

Apache 2.0 — Copyright 2026 Sirdata (sirdata.com).
