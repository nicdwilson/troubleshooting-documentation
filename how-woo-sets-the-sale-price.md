# WooCommerce Sale Price Display - Code Path Analysis

## Overview
This document traces the complete code path that ensures a sale price displays correctly on the product page when a sale goes live in WooCommerce.

## Key Components

### 1. Scheduled Sales System
**Location:** `includes/wc-product-functions.php` (lines 524-588)

WooCommerce uses a scheduled action system to activate/deactivate sales based on date ranges.

#### Hook Registration
- **Hook:** `woocommerce_scheduled_sales`
- **Function:** `wc_scheduled_sales()`
- **Scheduling:** Registered in `class-woocommerce.php` (line 1539)
  - Runs daily at midnight (site timezone)
  - Uses Action Scheduler: `as_schedule_recurring_action()`
  - First run: `strtotime('00:00 tomorrow ' . $offset_hours)`
  - Recurrence: `DAY_IN_SECONDS` (every 24 hours)

#### Process Flow

**A. Finding Products with Sales Starting:**
```php
$product_ids = $data_store->get_starting_sales();
```

**Location:** `includes/data-stores/class-wc-product-data-store-cpt.php` (lines 1331-1349)

**Query Logic:**
- Finds products where:
  - `_sale_price_dates_from` meta exists and > 0
  - `_sale_price_dates_from` < current timestamp (sale should have started)
  - `_price` != `_sale_price` (price hasn't been updated yet)

**B. Activating Sales:**
For each product with a starting sale:
1. Load product: `wc_get_product($product_id)`
2. Get sale price: `$product->get_sale_price()`
3. Update active price: `$product->set_price($sale_price)`
4. Clear start date: `$product->set_date_on_sale_from('')`
5. Save product: `$product->save()`
6. Clear transients: `delete_product_specific_transients()`
7. Clear sale cache: `delete_transient('wc_products_onsale')`

### 2. Product Save Process
**Location:** `includes/data-stores/class-wc-product-data-store-cpt.php` (lines 808-836)

When a product is saved, `handle_updated_props()` ensures the `_price` meta is synchronized:

```php
$product_price_props = array('date_on_sale_from', 'date_on_sale_to', 'regular_price', 'sale_price', 'product_type');
if (count(array_intersect($product_price_props, $this->updated_props)) > 0) {
    if ($product->is_on_sale('edit')) {
        update_post_meta($product->get_id(), '_price', $product->get_sale_price('edit'));
        $product->set_price($product->get_sale_price('edit'));
    } else {
        update_post_meta($product->get_id(), '_price', $product->get_regular_price('edit'));
        $product->set_price($product->get_regular_price('edit'));
    }
}
```

**Key Points:**
- Updates `_price` meta field based on `is_on_sale()` status
- This happens both during scheduled sales AND manual product saves
- The `_price` meta is the "active price" that gets displayed

### 3. Product Page Display

#### Template File
**Location:** `templates/single-product/price.php` (line 25)

```php
<p class="<?php echo esc_attr(apply_filters('woocommerce_product_price_class', 'price')); ?>">
    <?php echo $product->get_price_html(); ?>
</p>
```

#### Price HTML Generation
**Location:** `includes/abstracts/abstract-wc-product.php` (lines 1965-1975)

```php
public function get_price_html($deprecated = '') {
    if ('' === $this->get_price()) {
        $price = apply_filters('woocommerce_empty_price_html', '', $this);
    } elseif ($this->is_on_sale()) {
        $price = wc_format_sale_price(
            wc_get_price_to_display($this, array('price' => $this->get_regular_price())),
            wc_get_price_to_display($this)
        ) . $this->get_price_suffix();
    } else {
        $price = wc_price(wc_get_price_to_display($this)) . $this->get_price_suffix();
    }
    
    return apply_filters('woocommerce_get_price_html', $price, $this);
}
```

**Flow:**
1. Check if product has a price: `$this->get_price()`
2. Check if on sale: `$this->is_on_sale()`
3. If on sale: Format with strikethrough regular price + sale price
4. If not on sale: Format regular price only

### 4. Sale Status Check
**Location:** `includes/abstracts/abstract-wc-product.php` (lines 1709-1724)

```php
public function is_on_sale($context = 'view') {
    if ('' !== (string) $this->get_sale_price($context) 
        && $this->get_regular_price($context) > $this->get_sale_price($context)) {
        $on_sale = true;
        
        // Check sale start date
        if ($this->get_date_on_sale_from($context) 
            && $this->get_date_on_sale_from($context)->getTimestamp() > time()) {
            $on_sale = false;
        }
        
        // Check sale end date
        if ($this->get_date_on_sale_to($context) 
            && $this->get_date_on_sale_to($context)->getTimestamp() < time()) {
            $on_sale = false;
        }
    } else {
        $on_sale = false;
    }
    
    return 'view' === $context 
        ? apply_filters('woocommerce_product_is_on_sale', $on_sale, $this) 
        : $on_sale;
}
```

**Logic:**
1. Must have a sale price set
2. Sale price must be less than regular price
3. Current time must be >= sale start date (if set)
4. Current time must be <= sale end date (if set)
5. Filterable via `woocommerce_product_is_on_sale`

### 5. Price Formatting

#### Sale Price Format
**Location:** `includes/wc-formatting-functions.php` (lines 1350-1374)

```php
function wc_format_sale_price($regular_price, $sale_price) {
    $formatted_regular_price = is_numeric($regular_price) ? wc_price($regular_price) : $regular_price;
    $formatted_sale_price = is_numeric($sale_price) ? wc_price($sale_price) : $sale_price;
    
    // Strikethrough pricing
    $price = '<del aria-hidden="true">' . $formatted_regular_price . '</del> ';
    
    // Screen reader text for accessibility
    $price .= '<span class="screen-reader-text">';
    $price .= esc_html(sprintf(__('Original price was: %s.', 'woocommerce'), 
        wp_strip_all_tags($formatted_regular_price)));
    $price .= '</span>';
    
    // Add the sale price
    $price .= '<ins aria-hidden="true">' . $formatted_sale_price . '</ins>';
    
    // Screen reader text for sale price
    $price .= '<span class="screen-reader-text">';
    $price .= esc_html(sprintf(__('Current price is: %s.', 'woocommerce'), 
        wp_strip_all_tags($formatted_sale_price)));
    $price .= '</span>';
    
    return apply_filters('woocommerce_format_sale_price', $price, $regular_price, $sale_price);
}
```

#### Price to Display
**Location:** `includes/wc-product-functions.php` (lines 1322-1350)

```php
function wc_get_price_to_display($product, $args = array()) {
    $args = wp_parse_args($args, array(
        'qty' => 1,
        'price' => $product->get_price(),  // Uses active price from _price meta
        'display_context' => 'shop',
    ));
    
    $price = $args['price'];
    $tax_display = get_option(
        'cart' === $args['display_context'] 
            ? 'woocommerce_tax_display_cart' 
            : 'woocommerce_tax_display_shop'
    );
    
    return 'incl' === $tax_display
        ? wc_get_price_including_tax($product, array('qty' => $args['qty'], 'price' => $price))
        : wc_get_price_excluding_tax($product, array('qty' => $args['qty'], 'price' => $price));
}
```

**Key Points:**
- Uses `$product->get_price()` which returns the `_price` meta value
- Applies tax based on shop/cart display settings
- Handles quantity multipliers

## Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ 1. ADMIN SETS SALE PRICE WITH DATES                         │
│    - Sets _sale_price meta                                   │
│    - Sets _sale_price_dates_from meta                        │
│    - Sets _sale_price_dates_to meta                          │
│    - Sets _price to regular_price (sale not active yet)     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. SCHEDULED ACTION TRIGGERED (Daily at Midnight)           │
│    Hook: woocommerce_scheduled_sales                         │
│    Function: wc_scheduled_sales()                           │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. FIND PRODUCTS WITH STARTING SALES                        │
│    get_starting_sales()                                      │
│    - Query products where:                                   │
│      * _sale_price_dates_from < current_time                │
│      * _price != _sale_price (not yet activated)             │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. ACTIVATE SALES                                            │
│    For each product:                                         │
│    - $product->set_price($sale_price)                        │
│    - Updates _price meta to sale_price                        │
│    - $product->set_date_on_sale_from('')                     │
│    - $product->save()                                        │
│    - Clear transients                                        │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. PRODUCT PAGE LOAD                                         │
│    Template: single-product/price.php                        │
│    Calls: $product->get_price_html()                         │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. CHECK SALE STATUS                                         │
│    $product->is_on_sale()                                    │
│    - Checks sale_price exists                                │
│    - Checks sale_price < regular_price                       │
│    - Checks current time >= date_on_sale_from                │
│    - Checks current time <= date_on_sale_to                  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. FORMAT PRICE HTML                                         │
│    If on sale:                                               │
│    - wc_format_sale_price(regular, sale)                     │
│      * <del>regular_price</del> <ins>sale_price</ins>       │
│    If not on sale:                                           │
│    - wc_price(regular_price)                                 │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 8. DISPLAY ON PAGE                                           │
│    Rendered HTML with formatted prices                        │
└─────────────────────────────────────────────────────────────┘
```

## Key Meta Fields

1. **`_price`** - Active/current price (what gets displayed)
   - Set to `_sale_price` when sale is active
   - Set to `_regular_price` when sale is not active

2. **`_regular_price`** - Regular product price

3. **`_sale_price`** - Sale price value

4. **`_sale_price_dates_from`** - Unix timestamp when sale starts

5. **`_sale_price_dates_to`** - Unix timestamp when sale ends

## Important Notes

### Real-time vs Scheduled Updates

**Scheduled Updates (Primary Method):**
- Sales activate/deactivate via daily cron job
- Runs at midnight (site timezone)
- Updates `_price` meta field
- Clears product transients

**Real-time Checks (Fallback):**
- `is_on_sale()` checks dates in real-time
- If scheduled job hasn't run yet, date checks still work
- However, `get_price()` returns `_price` meta, which may not be updated

### Potential Timing Issues

If a sale is scheduled to start at 10:00 AM but the cron runs at midnight:
- Sale won't activate until next midnight run
- However, `is_on_sale()` will return `true` if current time >= sale start date
- But `get_price()` may still return regular price if `_price` meta wasn't updated

**Solution:** WooCommerce relies on the scheduled job, but `is_on_sale()` provides real-time validation.

### Cache Considerations

- Product transients are cleared when sales activate/deactivate
- `wc_products_onsale` transient is deleted
- Product-specific transients are cleared via `delete_product_specific_transients()`

## Filters & Hooks

### Available Filters

1. **`woocommerce_product_is_on_sale`** - Modify sale status
   - Location: `abstract-wc-product.php:1723`
   - Params: `$on_sale`, `$product`

2. **`woocommerce_get_price_html`** - Modify price HTML output
   - Location: `abstract-wc-product.php:1974`
   - Params: `$price`, `$product`

3. **`woocommerce_format_sale_price`** - Modify sale price formatting
   - Location: `wc-formatting-functions.php:1373`
   - Params: `$price`, `$regular_price`, `$sale_price`

### Available Actions

1. **`wc_before_products_starting_sales`** - Before sales activate
   - Params: `$product_ids`

2. **`wc_after_products_starting_sales`** - After sales activate
   - Params: `$product_ids`

3. **`wc_before_products_ending_sales`** - Before sales end
   - Params: `$product_ids`

4. **`wc_after_products_ending_sales`** - After sales end
   - Params: `$product_ids`

## Summary

The sale price display system works through:

1. **Scheduled Activation:** Daily cron job activates sales based on `_sale_price_dates_from`
2. **Price Meta Update:** `_price` meta is updated to `_sale_price` when sale activates
3. **Real-time Validation:** `is_on_sale()` checks dates in real-time for accuracy
4. **Template Rendering:** Product page template calls `get_price_html()`
5. **Conditional Formatting:** If `is_on_sale()` is true, formats with strikethrough regular price + sale price
6. **Tax Calculation:** `wc_get_price_to_display()` applies tax based on shop settings

The system ensures sale prices display correctly by:
- Updating the active price (`_price` meta) when sales start
- Validating sale status in real-time during page load
- Formatting prices appropriately based on sale status
- Clearing caches to ensure fresh data

