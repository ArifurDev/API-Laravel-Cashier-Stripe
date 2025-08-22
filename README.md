# Laravel Cashier Stripe Integration for API Projects

This guide explains how to integrate **Laravel Cashier** with **Stripe** in a Laravel API project to handle subscriptions, payments, and webhooks. It includes step-by-step instructions, code examples, and best practices for building a robust subscription system.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Setting Up Webhooks](#setting-up-webhooks)
- [Creating Products and Prices](#creating-products-and-prices)
- [API Routes](#api-routes)
- [Subscription Controller](#subscription-controller)
- [Validation](#validation)
- [Handling Webhooks](#handling-webhooks)
- [Testing](#testing)
- [Best Practices](#best-practices)
- [Additional Resources](#additional-resources)

## Prerequisites
- Laravel 11.x (or compatible version)
- Composer
- Stripe account with API keys
- Basic knowledge of Laravel, REST APIs, and Stripe
- Database (e.g., MySQL, PostgreSQL)
- Laravel Sanctum or Passport for API authentication

## Installation

1. **Install Laravel Cashier**:
   Install the Cashier package for Stripe using Composer:
   ```bash
   composer require laravel/cashier
   ```

2. **Publish Migrations**:
   Publish Cashier's migrations to create subscription-related tables:
   ```bash
   php artisan vendor:publish --tag="cashier-migrations"
   ```

3. **Run Migrations**:
   Migrate the database to create the necessary tables:
   ```bash
   php artisan migrate
   ```

## Configuration

### Billable Model
Add the `Billable` trait to your `User` model (or another billable model) in `app/Models/User.php`:

```php
use Laravel\Cashier\Billable;

class User extends Authenticatable
{
    use Billable;

    protected $fillable = [
        'name', 'email', 'password', 'stripe_id', 'trial_ends_at',
    ];
}
```

### Environment Variables
Add your Stripe API keys and webhook secret to the `.env` file:

```env
STRIPE_KEY=pk_test_xxxxxxxxxxxxxxxxxxxxxxxx
STRIPE_SECRET=sk_test_xxxxxxxxxxxxxxxxxxxxxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxxxxxxxxxxxxx
```

You can find these in your [Stripe Dashboard](https://dashboard.stripe.com) under **Developers > API Keys** and **Developers > Webhooks**.

### CSRF Protection
Exclude the Stripe webhook route from Laravel's CSRF protection in `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->validateCsrfTokens(except: [
        'stripe/*',
    ]);
})
```

## Setting Up Webhooks

Stripe webhooks are essential for handling asynchronous events like subscription updates or payment failures.

1. **Create a Webhook**:
   Use the Cashier artisan command to create a webhook in Stripe:
   ```bash
   php artisan cashier:webhook --url "https://your-domain.com/stripe/webhook"
   ```

2. **Required Webhook Events**:
   Ensure the webhook listens to the following events in the Stripe Dashboard:
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
   - `customer.updated`
   - `customer.deleted`
   - `payment_method.automatically_updated`
   - `invoice.payment_action_required`
   - `invoice.payment_succeeded`

3. **Webhook URL**:
   By default, Cashier responds to `/stripe/webhook`. Ensure this URL is publicly accessible and matches the URL configured in the Stripe Dashboard.

## Creating Products and Prices

Create products and prices in Stripe, either manually via the Stripe Dashboard or programmatically. Below is an example of creating a product and price programmatically:

```php
use Laravel\Cashier\Cashier;

$stripeProduct = Cashier::stripe()->products->create([
    'name' => 'Premium Subscription',
]);

$stripePrice = Cashier::stripe()->prices->create([
    'product' => $stripeProduct->id,
    'unit_amount' => 999, // $9.99 in cents
    'currency' => 'usd',
    'recurring' => ['interval' => 'month'],
]);
```

Store the `stripe_price_id` in your database (e.g., in a `products` table) for use in subscriptions.

## API Routes

Define API routes in `routes/api.php` for handling subscriptions, status checks, and success/cancel callbacks:

```php
use App\Http\Controllers\StripeSubscriptionController;
use App\Http\Controllers\PanelController;

Route::middleware('auth:api')->group(function () {
    Route::post('/v1/subscription/payment', [StripeSubscriptionController::class, 'subscribe']);
    Route::get('/v1/subscription/status', [StripeSubscriptionController::class, 'checkSubscription']);
});

Route::get('/v1/subscription/panels', [PanelController::class, 'index']);
Route::get('/subscription/success', [StripeSubscriptionController::class, 'success'])->name('subscription.success');
Route::get('/subscription/cancel', [StripeSubscriptionController::class, 'cancel'])->name('subscription.cancel');
```

Ensure API authentication is set up (e.g., using Laravel Sanctum).

## Subscription Controller

Create a controller (`app/Http/Controllers/StripeSubscriptionController.php`) to handle subscription-related logic:

```php
namespace App\Http\Controllers;

use App\Http\Requests\SubscribeRequest;
use App\Models\Product;
use Illuminate\Http\Request;
use Laravel\Cashier\Cashier;

class StripeSubscriptionController extends Controller
{
    /**
     * Create a new subscription.
     */
    public function subscribe(SubscribeRequest $request)
    {
        $validated = $request->validated();
        $product = Product::findOrFail($validated['product_id']);

        $user = auth()->user();
        try {
            $checkoutSession = $user->newSubscription('default', $product->stripe_price_id)
                ->allowPromotionCodes()
                ->checkout([
                    'success_url' => route('subscription.success') . '?session_id={CHECKOUT_SESSION_ID}',
                    'cancel_url' => route('subscription.cancel'),
                ]);

            return response()->json([
                'success' => true,
                'checkout_url' => $checkoutSession->url,
                'message' => 'Checkout session created',
            ], 200);
        } catch (\Exception $e) {
            return response()->json([
                'success' => false,
                'message' => 'Failed to create checkout session: ' . $e->getMessage(),
            ], 400);
        }
    }

    /**
     * Handle successful subscription.
     */
    public function success(Request $request)
    {
        // Optionally verify the checkout session using $request->session_id
        return response()->json([
            'success' => true,
            'message' => 'Subscription created successfully',
        ], 200);
    }

    /**
     * Handle canceled subscription attempt.
     */
    public function cancel()
    {
        return response()->json([
            'success' => false,
            'message' => 'Subscription cancelled',
        ], 200);
    }

    /**
     * Check if the user has an active subscription.
     */
    public function checkSubscription()
    {
        $user = auth()->user();
        $isSubscribed = $user->subscribed('default');

        return response()->json([
            'success' => true,
            'is_subscribed' => $isSubscribed,
            'subscription' => $isSubscribed ? $user->subscription('default')->only([
                'stripe_id',
                'stripe_status',
                'product_id',
                'ends_at',
            ]) : null,
            'message' => 'Subscription status retrieved',
        ], 200);
    }
}
```

## Validation

Create a `FormRequest` class for validating subscription requests in `app/Http/Requests/SubscribeRequest.php`:

```php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class SubscribeRequest extends FormRequest
{
    public function authorize()
    {
        return auth()->check();
    }

    public function rules()
    {
        return [
            'product_id' => 'required|exists:products,id',
        ];
    }
}
```

## Handling Webhooks

Cashier’s default webhook controller (`/stripe/webhook`) handles most subscription events. To customize webhook handling, listen for Cashier’s events in a service provider or event listener:

```php
use Illuminate\Support\Facades\Event;
use Laravel\Cashier\Events\WebhookReceived;

Event::listen(WebhookReceived::class, function (WebhookReceived $event) {
    if ($event->payload['type'] === 'invoice.payment_succeeded') {
        // Handle successful payment (e.g., update user records, send notifications)
    }
});
```

## Testing

1. **Local Webhook Testing**:
   Use the Stripe CLI to simulate webhook events:
   ```bash
   stripe listen --forward-to https://your-domain.com/stripe/webhook
   ```

2. **API Testing**:
   Use tools like Postman to test your API endpoints. Test with Stripe’s test card numbers (e.g., `4242 4242 4242 4242`).

3. **Unit Testing**:
   Write tests using PHPUnit or Laravel Dusk to verify subscription creation, status checks, and webhook handling.

## Best Practices

- **Secure Webhooks**: Always verify webhook signatures using the `STRIPE_WEBHOOK_SECRET`.
- **Error Handling**: Catch and handle Stripe exceptions (e.g., `Stripe\Exception\CardException`) in your controller.
- **Idempotency**: Ensure webhook handlers are idempotent to avoid duplicate processing.
- **Logging**: Log webhook events and errors for debugging.
- **Rate Limiting**: Apply rate limiting to subscription endpoints to prevent abuse.
- **API Responses**: Use Laravel’s API resources for consistent JSON responses.
- **Grace Periods**: Configure grace periods for subscription cancellations or payment failures in the Cashier configuration.

## Additional Resources

- **Official Laravel Cashier Documentation**: [https://laravel.com/docs/cashier-stripe](https://laravel.com/docs/cashier-stripe)
- **Stripe API Documentation**: [https://docs.stripe.com/api](https://docs.stripe.com/api)
- **Stripe Webhook Documentation**: [https://docs.stripe.com/webhooks](https://docs.stripe.com/webhooks)
- **Laravel API Resource Documentationavic**: [https://laravel.com/docs/eloquent-resources](https://laravel.com/docs/eloquent-resources)
- **Laravel Daily Tutorials**: [https://laraveldaily.com/post/stripe-payments-in-laravel](https://laraveldaily.com/post/stripe-payments-in-laravel)
- **GitHub Repository**: [https://github.com/laravel/cashier-stripe](https://github.com/laravel/cashier-stripe)

---

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
