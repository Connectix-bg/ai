# Platforms reference

## Official plugins - extend, do not rebuild

| Platform | Plugin | What it already does | Extend it when |
|---|---|---|---|
| WordPress / WooCommerce | Connectix for WooCommerce (wordpress.org and the console's Integrations page) | Order status notifications (Viber -> SMS fallback), templates per status, blacklist opt-out, shipment tracking hooks | The user needs a new trigger (custom order status, abandoned cart) - add a WooCommerce action hook that calls the plugin's sender, or a filter on its template parameters |
| PrestaShop | Connectix PrestaShop module | Order state notifications, template mapping, OTP at checkout (optional) | New hook: register a PrestaShop hook in the module and call the module's message service |
| OpenCart | Connectix OpenCart extension | Order history notifications, template mapping | New event: add an OpenCart event listener that reuses the extension's API client |

Rules of thumb:

1. Check the console's Integrations page or ask the user whether the official plugin is installed
   before writing any shop code.
2. The plugins already hold the token, the host switch and the template mapping in their settings
   page. Never hard-code a token in a theme or a custom module.
3. Custom code on these platforms should call the plugin's own client/service class so that the
   sandbox/live switch stays in one place.
4. Cloud platforms with a built-in Connectix connector (for example CloudCart) need no code: the
   connector is enabled in the platform's admin.

## Everything else - HTTP API

| Stack | Approach |
|---|---|
| PHP (Laravel, Symfony, plain) | The official PHP SDK (`connectix/php-sdk`, Composer) or Guzzle/Symfony HttpClient against the sandbox. Read the base URL and the token from env (`CONNECTIX_BASE_URL=https://api-sandbox.connectix.bg`, `CONNECTIX_SANDBOX_TOKEN`) |
| Node / TypeScript | `fetch` with the three headers from SKILL.md section 2; a thin client module with one method per operationId |
| Python | `httpx`/`requests`; same client shape |
| Magento 2, Shopify, custom e-commerce | Register an integration (`registerIntegration`) and hooks (`registerIntegrationHook`), send order messages with `sendFallbackMessage`, push shipments with `createShipment` |
| Automation tools (Make, n8n, Zapier) | Plain HTTP module; `registerIntegration` with `type: automation` |

## Integration checklist for a shop

1. `getMe` to confirm the sandbox token.
2. `listTemplates`; map order statuses to approved template ids in configuration, not code.
3. `registerIntegration` (`type`, `platform`, `secret`) and store the returned `token`.
4. `registerIntegrationHook` for `callback` (status), `blacklist` (opt-outs) and, if shipments are
   tracked, `tracking`.
5. On order status change: `checkBlacklist` for promotional sends only; `sendFallbackMessage`
   (Viber step 1, SMS step 2) for transactional ones.
6. On shipment creation: `createShipment` with `?integration=<token>`, `trackingNumber` and `phone`.
7. Tests against the sandbox; go-live is the user's step.
