# Resubscribe Button Code Path Analysis

## Overview
This document traces the code path that determines when the "Resubscribe" button appears in the customer detailed subscription view on the My Account page in WooCommerce Subscriptions.

## Code Path Flow

### 1. Template Entry Point
**File:** `templates/myaccount/view-subscription.php`

The subscription details view is rendered through this template, which calls:
```php
do_action( 'woocommerce_subscription_details_table', $subscription );
```

### 2. Subscription Details Template
**File:** `templates/myaccount/subscription-details.php` (Lines 77-101)

The subscription details table template displays the actions:
```php
<?php $actions = wcs_get_all_user_actions_for_subscription( $subscription, get_current_user_id() ); ?>
<?php if ( ! empty( $actions ) ) : ?>
    <tr>
        <td><?php esc_html_e( 'Actions', 'woocommerce-subscriptions' ); ?></td>
        <td>
            <?php foreach ( $actions as $key => $action ) : ?>
                <a href="<?php echo esc_url( $action['url'] ); ?>" class="...">
                    <?php echo esc_html( $action['name'] ); ?>
                </a>
            <?php endforeach; ?>
        </td>
    </tr>
<?php endif; ?>
```

### 3. Action Generation Function
**File:** `includes/core/wcs-user-functions.php` (Lines 301-336)

The function `wcs_get_all_user_actions_for_subscription()` generates available actions:

```php
function wcs_get_all_user_actions_for_subscription( $subscription, $user_id ) {
    $actions = array();

    if ( user_can( $user_id, 'edit_shop_subscription_status', $subscription->get_id() ) ) {
        $current_status = $subscription->get_status();

        // Reactivate button (shown first if applicable)
        if ( $subscription->can_be_updated_to( 'active' ) && ! $subscription->needs_payment() ) {
            $actions['reactivate'] = array( ... );
        }

        // RESUBSCRIBE BUTTON CONDITION (Line 316)
        if ( wcs_can_user_resubscribe_to( $subscription, $user_id ) && false == $subscription->can_be_updated_to( 'active' ) ) {
            $actions['resubscribe'] = array(
                'url'      => wcs_get_users_resubscribe_link( $subscription ),
                'name'     => __( 'Resubscribe', 'woocommerce-subscriptions' ),
                'block_ui' => true,
            );
        }

        // Cancel button
        if ( $subscription->can_be_updated_to( 'cancelled' ) && ... ) {
            $actions['cancel'] = array( ... );
        }
    }

    return apply_filters( 'wcs_view_subscription_actions', $actions, $subscription, $user_id );
}
```

## Conditions for Resubscribe Button to Appear

The resubscribe button appears when **ALL** of the following conditions are met:

### Condition 1: User Capability Check
```php
user_can( $user_id, 'edit_shop_subscription_status', $subscription->get_id() )
```
- The user must have the capability to edit the subscription status
- This is checked at the beginning of the action generation function

### Condition 2: Cannot Be Reactivated
```php
false == $subscription->can_be_updated_to( 'active' )
```
- The subscription **cannot** be updated to 'active' status
- This ensures the "Reactivate" button takes precedence when reactivation is possible
- If `can_be_updated_to('active')` returns `true`, the "Reactivate" button is shown instead

### Condition 3: User Can Resubscribe
```php
wcs_can_user_resubscribe_to( $subscription, $user_id )
```
- This is the main validation function that checks multiple sub-conditions

## Detailed Analysis of `wcs_can_user_resubscribe_to()`

**File:** `includes/core/wcs-resubscribe-functions.php` (Lines 161-232)

This function returns `false` if **ANY** of the following conditions are true:

### Sub-condition 1: Subscription Object Invalid
```php
if ( empty( $subscription ) ) {
    $can_user_resubscribe = false;
}
```

### Sub-condition 2: User Lacks Permission
```php
if ( ! user_can( $user_id, 'subscribe_again', $subscription->get_id() ) ) {
    $can_user_resubscribe = false;
}
```
- User must have the `subscribe_again` capability for this subscription

### Sub-condition 3: Subscription Status Not Inactive
```php
if ( ! $subscription->has_status( array( 'pending-cancel', 'cancelled', 'expired', 'trash' ) ) ) {
    $can_user_resubscribe = false;
}
```
- Subscription must be in one of these inactive statuses:
  - `pending-cancel`
  - `cancelled`
  - `expired`
  - `trash`

### Sub-condition 4: Total Amount is Zero or Negative
```php
if ( $subscription->get_total() <= 0 ) {
    $can_user_resubscribe = false;
}
```
- Prevents resubscribing to subscriptions with $0 total (to avoid circumventing sign-up fees)

### Sub-condition 5: Contains Unavailable Products
```php
if ( $subscription->contains_unavailable_product() ) {
    $can_user_resubscribe = false;
}
```
- All products in the subscription must still be available/purchasable

### Sub-condition 6: Final Validation Check
If all previous checks pass, the function performs a final validation:

```php
$resubscribe_order_ids = $subscription->get_related_orders( 'ids', 'resubscribe' );

// Check if all line items still exist
$all_line_items_exist = true;
$has_active_limited_subscription = false;

foreach ( $subscription->get_items() as $line_item ) {
    $product = ( ! empty( $line_item['variation_id'] ) ) 
        ? wc_get_product( $line_item['variation_id'] ) 
        : wc_get_product( $line_item['product_id'] );

    if ( false === $product ) {
        $all_line_items_exist = false;
        break;
    }

    // Check for limited subscriptions (one active subscription per product)
    if ( 'active' === wcs_get_product_limitation( $product ) ) {
        $limited_product_id = $product->is_type( 'variation' ) 
            ? $product->get_parent_id() 
            : $product->get_id();

        if ( wcs_user_has_subscription( $user_id, $limited_product_id, 'on-hold' ) 
          || wcs_user_has_subscription( $user_id, $limited_product_id, 'active' ) ) {
            $has_active_limited_subscription = true;
            break;
        }
    }
}

// Final check
if ( empty( $resubscribe_order_ids ) 
  && $subscription->get_payment_count() > 0 
  && true === $all_line_items_exist 
  && false === $has_active_limited_subscription ) {
    $can_user_resubscribe = true;
} else {
    $can_user_resubscribe = false;
}
```

**Final validation requires:**
1. **No existing resubscribe orders**: `empty( $resubscribe_order_ids )`
   - Prevents showing resubscribe button if a resubscribe order already exists
   - Ensures the subscription hasn't already been resubscribed to

2. **At least one payment made**: `$subscription->get_payment_count() > 0`
   - Ensures the subscription had at least one successful payment
   - Prevents circumventing sign-up fees

3. **All line items exist**: `true === $all_line_items_exist`
   - All products/variations in the subscription must still exist in the store

4. **No active limited subscription**: `false === $has_active_limited_subscription`
   - For products with limitation set to 'active', user cannot have an active or on-hold subscription to the same product
   - Prevents multiple active subscriptions for limited products

### Filter Hook
The result can be modified by other plugins:
```php
return apply_filters( 'wcs_can_user_resubscribe_to_subscription', $can_user_resubscribe, $subscription, $user_id );
```

## Understanding `can_be_updated_to('active')`

**File:** `includes/core/class-wc-subscription.php` (Lines 337-426)

A subscription can be updated to 'active' status if it meets one of these conditions:

1. **On-hold with reactivation support**:
   - Payment method supports `subscription_reactivation`
   - Subscription status is `on-hold`
   - End date is not in the past (if set)

2. **Pending status**:
   - Subscription status is `pending`

3. **Pending-cancel with conditions**:
   - Subscription status is `pending-cancel`
   - End date is in the future
   - Either:
     - Subscription is manual, OR
     - Payment method doesn't support gateway scheduled payments AND supports date changes AND supports reactivation

## Summary: When Resubscribe Button Appears

The resubscribe button appears when:

1. ✅ User has `edit_shop_subscription_status` capability
2. ✅ Subscription **cannot** be reactivated (`can_be_updated_to('active')` returns `false`)
3. ✅ User has `subscribe_again` capability
4. ✅ Subscription status is one of: `pending-cancel`, `cancelled`, `expired`, or `trash`
5. ✅ Subscription total is greater than $0
6. ✅ All products in subscription are still available
7. ✅ No resubscribe order already exists for this subscription
8. ✅ Subscription had at least one payment (`payment_count > 0`)
9. ✅ All line items (products/variations) still exist
10. ✅ User doesn't have an active/on-hold subscription to a limited product (if applicable)
11. ✅ Filter `wcs_can_user_resubscribe_to_subscription` allows it (default: true if all above pass)

## Key Differences: Reactivate vs Resubscribe

- **Reactivate Button**: Shown when subscription can be updated to 'active' status (e.g., on-hold, pending-cancel with future end date)
- **Resubscribe Button**: Shown when subscription cannot be reactivated (e.g., cancelled, expired) but can be resubscribed to by creating a new subscription order

## Related Files

- `templates/myaccount/view-subscription.php` - Main subscription view template
- `templates/myaccount/subscription-details.php` - Subscription details table with actions
- `includes/core/wcs-user-functions.php` - Action generation function
- `includes/core/wcs-resubscribe-functions.php` - Resubscribe validation logic
- `includes/core/class-wc-subscription.php` - Subscription class with `can_be_updated_to()` method
