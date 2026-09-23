# Shopify Product Feed Manager

Automated product feed generator for Shopify stores that creates variant-level XML feeds for Google Shopping, Meta (Facebook), and other advertising channels.

## Features

✅ **Variant-Level Inventory** - Each product variant (size/color) is a separate feed entry with accurate stock status
✅ **Free Hosting** - Feeds hosted on GitHub Pages at no cost
✅ **Multi-Channel Support** - Google Shopping and Meta feeds out of the box
✅ **Automated Scheduling** - GitHub Actions workflow runs every 6 hours
✅ **Configurable Mappings** - Easy YAML configuration for field mappings
✅ **API 2025-10** - Uses latest Shopify Admin API version

## Why This Matters

**Problem**: Third-party feed services like DataFeedWatch charge monthly fees and often show products as "in stock" when only unpopular sizes are available.

**Solution**: This tool generates feeds at the variant level, so:
- Only truly available products show as "in stock"
- No monthly fees for feed generation
- Full control over feed customization
- Feeds hosted on your Shopify CDN (free)

## Setup

### 1. Install Dependencies

```bash
pip install -r requirements.txt
```

### 2. Configure Your Store

Edit `config.yaml`. Shopify app credentials are shared across all stores (one app, installed on each store) and set once at the top level; each store just needs its own domains/locale:

```yaml
client_id: ${SHOPIFY_CLIENT_ID}
client_secret: ${SHOPIFY_CLIENT_SECRET}

stores:
  - name: FR
    display_name: France
    shop_domain: your-store.myshopify.com
    customer_domain: your-store.com
    language: fr
    currency: EUR
```

**Get Shopify OAuth Credentials:**
1. Go to [Shopify Partners](https://partners.shopify.com) and log in (or create an account)
2. Click "Apps" → "Create app"
3. Choose "Create app manually" and give it a name (e.g., "Feed Generator")
4. In your app settings, go to "Configuration" → "Admin API access scopes"
5. Enable `read_products` scope and save
6. Copy the **Client ID** and **Client secret** from "Client credentials"
7. Install the app on each store (all stores share the same app/credentials)
8. Add `SHOPIFY_CLIENT_ID` and `SHOPIFY_CLIENT_SECRET` as GitHub Actions secrets (Settings → Secrets and variables → Actions) — no per-store suffix needed

> ⚠️ **Note**: As of January 2026, Shopify uses OAuth client credentials instead of permanent tokens. The feed generator automatically obtains short-lived access tokens (valid 24 hours) using your client credentials.

### 3. Customize Channel Mappings (Optional)

Edit `channel_mappings.yaml` to customize which Shopify fields map to feed fields:

```yaml
channels:
  google:
    fields:
      - id: variant.id
      - item_group_id: id
      - title: title
      - price: "{variant.price} {currency}"
      - availability: availability
      # ... more fields
```

**Template Syntax:**
- `variant.price` - Gets variant price
- `{variant.price} {currency}` - Template with placeholders
- `'new'` - Static string value
- `images[0].src` - Nested array/object access

## Usage

### Generate Feeds Locally

```bash
python generate_feeds.py
```

Feeds are saved to `feeds/` and `docs/` (for GitHub Pages).

### Automated Scheduling with GitHub Actions

The included workflow (`.github/workflows/generate-feeds.yml`) automatically:
- Runs every 6 hours
- Generates fresh feeds
- Commits to `docs/` for GitHub Pages hosting
- Archives feed copies as artifacts

**To enable:**
1. Push this code to a GitHub repository
2. GitHub Actions will run automatically
3. Check Actions tab for status

**Manual trigger:**
- Go to Actions → "Generate and Upload Product Feeds" → "Run workflow"

## Feed Structure

### Variant-Level Feeds

Each product variant becomes a separate item:

```xml
<item>
  <g:id>49802838671706</g:id>                      <!-- Variant ID -->
  <g:item_group_id>9877227897178</g:item_group_id> <!-- Product ID (groups variants) -->
  <g:title>bisgaard aarhus rain jacket caramel</g:title>
  <g:link>https://your-store.com/products/jacket?variant=49802838671706</g:link>
  <g:price>69.95 EUR</g:price>
  <g:availability>in stock</g:availability>         <!-- Accurate per-variant stock -->
  <g:size>4Y</g:size>
  <g:color>caramel</g:color>
</item>
```

### Inventory Logic

Availability is decided in two layers:

1. **Per-variant**, same as before — does this specific size have stock?

```python
# generate_feeds.py:321-334
def calculate_availability(variant):
    inventory_qty = variant.get('inventory_quantity', 0)
    inventory_policy = variant.get('inventory_policy', 'deny')

    if inventory_policy == 'continue':
        return 'preorder'  # Can sell when out of stock

    return 'in stock' if inventory_qty > 0 else 'out of stock'
```

2. **Per-product hard cap** — a product only advertises at all once at least `min_sizes_in_stock` (config.yaml, default `3`) of its sizes are sellable. Below that, every variant of the product reports `out of stock` in the ad feed, regardless of its own availability. This exists because a thin size run (e.g. the last 1-2 sizes left out of a full run) tends to draw ad clicks that don't convert — shoppers' size is often already gone. Products with fewer total variants than the threshold (e.g. single-variant items) are exempt and use plain per-variant availability.

```python
# generate_feeds.py:336-356
def product_meets_stock_cap(product, min_sizes_in_stock):
    variants = product.get('variants', [])
    if len(variants) <= min_sizes_in_stock:
        return True
    sellable = sum(1 for v in variants if calculate_availability(v) in ('in stock', 'preorder'))
    return sellable >= min_sizes_in_stock
```

## File Structure

```
Feed Manager/
├── generate_feeds.py          # Main feed generator
├── config.yaml                # Store credentials & settings
├── channel_mappings.yaml      # Field mapping configuration
├── requirements.txt           # Python dependencies
├── .github/
│   └── workflows/
│       └── generate-feeds.yml # Automated scheduling
└── feeds/                     # Generated XML files
    └── FR/
        ├── google_fr_EUR.xml
        └── meta_fr_EUR.xml
```

## Adding More Stores

Since all stores share one Shopify app/credential pair, adding a store is a single edit to `config.yaml` — add one block to the `stores:` list:

```yaml
stores:
  - name: NL
    display_name: Netherlands
    shop_domain: store-nl.myshopify.com
    customer_domain: store-nl.com
    language: nl
    currency: EUR
```

That's it — no new GitHub Secrets and no workflow changes needed (`SHOPIFY_CLIENT_ID`/`SHOPIFY_CLIENT_SECRET` already apply to every store). `docs/index.html`'s feed listing is auto-generated from this same store list on every run (between the `STORES_START`/`STORES_END` markers in its `<script>` block), so it never needs manual editing either.

For local development, `config.local.yaml` doesn't need a `stores:` list at all — `load_config()` merges it over `config.yaml`, so the store list is inherited automatically. Only add a `stores:` override there if you want to test against a subset of stores locally.

Feeds will be generated for each store automatically.

## Using Feeds in Ad Platforms

### Google Merchant Center

1. Get your GitHub Pages feed URL (e.g., `https://username.github.io/repo/EN_google_en_EUR.xml.gz`)
2. In Merchant Center → Products → Feeds → Add feed
3. Choose "Scheduled fetch" and paste the URL
4. Set fetch schedule (daily recommended)

### Meta Commerce Manager

1. Get your GitHub Pages feed URL (e.g., `https://username.github.io/repo/EN_meta_en_EUR.xml.gz`)
2. In Commerce Manager → Catalog → Data Sources
3. Add data feed with the Meta feed URL
4. Set update frequency

## Troubleshooting

### Feeds not generating

- Check Shopify access token has `read_products` scope
- Verify API version is 2025-10 or later
- Check your token hasn't expired

### Missing product fields

- Check if field exists in Shopify: Add `print(product)` in generate_feeds.py:248
- Update `channel_mappings.yaml` with correct field path
- Use `variant.field_name` for variant-specific fields

### GitHub Actions failing

- Ensure repository secrets are set (if using secrets instead of config.yaml)
- Check Actions logs for specific error messages
- Verify Python 3.11 compatibility

## Customization

### Add New Channels

Edit `channel_mappings.yaml`:

```yaml
channels:
  google:
    # ... existing
  meta:
    # ... existing
  new_channel:
    fields:
      - id: variant.id
      - title: title
      # ... your mappings
```

### Change Upload Frequency

Edit `.github/workflows/generate-feeds.yml`:

```yaml
schedule:
  - cron: '0 */4 * * *'  # Every 4 hours instead of 6
```

### Add Product Filters

Edit `generate_feeds.py:32`:

```python
# Example: Only products with specific tag
return [p for p in all_products if 'sale' in p.get('tags', '').lower()]
```

## Cost Savings

Replacing DataFeedWatch (~€79/month) + Confect.io (~€29/month):
- **Annual savings**: ~€1,296
- **This solution**: Free (using GitHub Pages for hosting)

## Support

For issues or questions:
- Check the troubleshooting section above
- Review Shopify Admin API docs: https://shopify.dev/docs/api/admin
- Open an issue in this repository

## License

MIT License - Use freely for commercial purposes
