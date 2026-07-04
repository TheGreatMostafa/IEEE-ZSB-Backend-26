# Task 7 - Blade, ORM, and the Design Patterns That Finally Made Sense

this one's less "here's a new feature" and more "here's four ideas i kept hearing about and finally understand." gonna go through each one the way i'd explain it to myself a week ago.

---

## Blade — why the views don't look like normal PHP anymore

first thing i noticed switching to Laravel: the view files don't have `<?php ?>` tags jammed everywhere. they use stuff like `{{ }}` and `@foreach` instead. turns out this is Blade, Laravel's templating engine, and it's not actually running that syntax directly — it's translating it first.

```
resources/views/products/index.blade.php   ← what i write
                    ↓ compiled the first time it's requested
storage/framework/views/7c2ad9.php          ← what actually runs
```

so the first person to visit a page pays a tiny compile cost, and literally everyone after that just hits the already-compiled plain PHP file. if i never touch the blade file again, it never recompiles again either. that explains why editing a view sometimes feels like it "didn't update" — you're looking at a stale cached file if something's misconfigured.

what actually gets compiled:

```blade
{{-- what i write --}}
<h1>{{ $product->name }}</h1>
```

```php
<?php
// what it turns into
<h1><?php echo e($product->name); ?></h1>
?>
```

that `e()` wrapper is doing `htmlspecialchars()` under the hood. so `{{ }}` isn't just shorthand for echo, it's echo-with-safety-built-in. if someone's product name field somehow had `<script>...</script>` in it, this just prints it as text on the page instead of running it. you'd have to go out of your way and use `{!! !!}` instead to turn that off, which is a good default honestly — nobody has to remember to escape things, you'd have to remember to *not* escape.

directives (`@if`, `@foreach`, etc.) are just the PHP control structures wearing a nicer outfit:

```blade
@foreach ($products as $product)
    <div class="card">
        <h3>{{ $product->name }}</h3>
        @if ($product->stock === 0)
            <span>sold out</span>
        @endif
    </div>
@endforeach
```

same thing written the old way:

```php
<?php foreach ($products as $product): ?>
    <div class="card">
        <h3><?= htmlspecialchars($product->name) ?></h3>
        <?php if ($product->stock === 0): ?>
            <span>sold out</span>
        <?php endif; ?>
    </div>
<?php endforeach; ?>
```

both do the exact same thing. i just don't have to keep switching my brain between "html mode" and "php mode" reading the top one.

the other big thing is layouts. instead of copy-pasting the header/nav/footer into every view:

```blade
{{-- layouts/storefront.blade.php --}}
<html>
<head><title>@yield('title')</title></head>
<body>
    @include('partials.header')
    @yield('content')
</body>
</html>
```

```blade
{{-- products/index.blade.php --}}
@extends('layouts.storefront')

@section('title', 'All Products')

@section('content')
    <h1>Shop</h1>
@endsection
```

any page that does `@extends('layouts.storefront')` just slots its content into that one `@yield('content')` spot. change the header once, every page using this layout gets it. i don't have to hunt down 15 files to fix a typo in the nav bar anymore.

---

## the ORM — stop writing SQL for every little thing

before this i was writing raw PDO everywhere. `prepare()`, `execute()`, `fetchAll()`, repeated in every controller. Eloquent (Laravel's ORM) replaces most of that with plain PHP method calls.

```php
<?php
// the old way
$stmt = $pdo->prepare("SELECT * FROM products WHERE category_id = ? AND in_stock = ?");
$stmt->execute([$categoryId, true]);
$products = $stmt->fetchAll(PDO::FETCH_CLASS, 'Product');

// with eloquent
$products = Product::where('category_id', $categoryId)
    ->where('in_stock', true)
    ->get();
?>
```

same query, way less to mess up. no string to typo, no manual fetch mode juggling.

relationships are the part that actually sold me on it though. instead of writing a JOIN every time i need a customer's orders:

```php
<?php
class Customer extends Model
{
    public function orders()
    {
        return $this->hasMany(Order::class);
    }
}
```

and then i just... use it like a normal property:

```php
<?php
$customer = Customer::find(12);

foreach ($customer->orders as $order) {
    echo $order->total;
}
```

no join, no manual mapping. define the relationship once in the model, use `->orders` anywhere forever.

and none of this cares what database is actually running behind it. same `Product::where(...)` code works whether `config/database.php` points at mysql, postgres, or sqlite. i think about *what* i want, eloquent figures out *how* to ask the specific database for it.

bonus stuff that comes free: `save()` handles the insert/update and stamps `created_at`/`updated_at` for you, and mass-assignment protection (`$fillable`) stops randos from setting fields they shouldn't be able to touch through a form.

```php
<?php
$order = new Order();
$order->customer_id = $customerId;
$order->total = $cartTotal;
$order->save(); // insert + timestamps, handled
?>
```

---

## Facades — why `Route::get()` isn't actually a static method

this one confused me at first because `Storage::put(...)` LOOKS like a static method call on a class called `Storage`. but `Storage` doesn't actually contain any file-handling code. it's a proxy.

```php
<?php
// what i write
Storage::put('avatars/user1.png', $imageData);

// what actually happens behind that
$disk = app()->make('filesystem');
$disk->put('avatars/user1.png', $imageData);
?>
```

so `Storage::put()` is really "go grab the real filesystem service out of the container, and call `put()` on that." the facade is just a friendly nickname pointing at something registered elsewhere. that's the whole Facade pattern — you get one simple call, and a much bigger, more annoying system sits behind it that you never have to think about.

`Mail::to($customer->email)->send(new OrderShipped($order));` is the same deal — behind that one line is a transport driver, a queue, and template rendering, but i only ever have to write that one line to use it.

why bother with this instead of just calling the real class directly? because i'd otherwise have to know how to resolve `filesystem` or `mailer` out of the container every single time, remember the exact binding key, etc. the facade just gives me a short name to remember instead.

---

## Factories — generating fake data without losing my mind

this one's specifically for testing and seeding. instead of manually typing out 30 fake products to test something, i define what a "normal" product looks like once:

```php
<?php
// database/factories/ProductFactory.php
class ProductFactory extends Factory
{
    public function definition(): array
    {
        return [
            'name'  => $this->faker->words(3, true),
            'price' => $this->faker->randomFloat(2, 5, 200),
            'stock' => $this->faker->numberBetween(0, 100),
        ];
    }
}
```

and then just... ask for however many i want:

```php
<?php
Product::factory()->create();          // one fake product
Product::factory()->count(30)->create(); // 30 of them, one line
```

the calling code doesn't need to know anything about how a product gets built — price ranges, faker logic, none of it. it just says "give me a product" and gets one back. that's the actual point of the Factory pattern, generalized beyond just testing: hide the messy construction logic behind a method that just hands you the finished object.

factories can also have "states" — variations on the base definition, without duplicating the whole thing:

```php
<?php
public function onSale(): static
{
    return $this->state(fn (array $attributes) => [
        'is_on_sale' => true,
        'price' => $attributes['price'] * 0.7,
    ]);
}

// usage
$discounted = Product::factory()->onSale()->create();
```

still don't have to know how a sale product is different internally, i just ask for one "on sale."

---

## SOLID — the five rules i kept seeing in every article about "clean code"

i'm not going to pretend i fully internalized all five of these overnight, but here's what actually clicked for each one, with a tiny example.

**S — single responsibility.** a class should only have one reason to change. i was originally shoving save/pdf/email logic all into one `Order` class:

```php
<?php
class Order
{
    public function save() { /* db stuff */ }
    public function generateInvoicePdf() { /* pdf stuff */ }
    public function notifyCustomer() { /* email stuff */ }
}
```

split apart, each job gets its own class:

```php
<?php
class Order { public function save() { /* just saves */ } }
class InvoicePdfGenerator { public function generate(Order $order) { /* just pdf */ } }
class OrderNotifier { public function notify(Order $order) { /* just email */ } }
```

now if the invoice layout changes, i only touch `InvoicePdfGenerator`. nothing else even knows that class exists.

**O — open/closed.** you should be able to add new behavior without editing old, already-working code. i had this:

```php
<?php
function shippingCost($method, $weight) {
    if ($method === 'standard') return $weight * 0.5;
    if ($method === 'express')  return $weight * 1.2;
}
```

every new shipping method meant editing this function again and risking breaking the existing ones. instead:

```php
<?php
interface ShippingMethod { public function cost($weight); }
class StandardShipping implements ShippingMethod {
    public function cost($weight) { return $weight * 0.5; }
}
class OvernightShipping implements ShippingMethod {
    public function cost($weight) { return $weight * 2.0; }
}
```

adding overnight shipping is a brand new class. `StandardShipping` never gets touched, never risks breaking.

**L — liskov substitution.** any subclass should be able to stand in for its parent without weird surprises. classic example that finally made this click for me was Rectangle/Square:

```php
<?php
class Rectangle {
    public function setWidth($w) { $this->width = $w; }
    public function setHeight($h) { $this->height = $h; }
}
class Square extends Rectangle {
    public function setWidth($w) { $this->width = $this->height = $w; } // sneaky
}
```

code that expects a `Rectangle` and calls `setWidth()` then `setHeight()` separately gets weird, wrong results if it's secretly holding a `Square`. the fix is to stop pretending square IS-A rectangle in code terms and just give both their own independent `area()` implementation off a shared `Shape` interface, so nothing assumes width and height move independently.

**I — interface segregation.** don't force a class to implement methods it'll never use.

```php
<?php
interface Printer {
    public function printDocument($file);
    public function scanDocument($file);
    public function faxDocument($file);
}
```

my cheap inkjet printer can't scan or fax. under this interface it'd have to fake those methods just to compile. split into smaller interfaces (`CanPrint`, `CanScan`) and it only implements what it can actually do.

**D — dependency inversion.** high-level code shouldn't be locked to one specific low-level implementation.

```php
<?php
class PaymentProcessor {
    private $gateway;
    public function __construct() {
        $this->gateway = new StripeGateway(); // stuck with stripe forever
    }
}
```

vs depending on an interface instead of a concrete class:

```php
<?php
interface PaymentGateway { public function charge($amount); }
class PaymentProcessor {
    public function __construct(private PaymentGateway $gateway) {}
}
```

now swapping stripe for paypal, or plugging in a fake gateway for tests, doesn't touch `PaymentProcessor` at all — the container just hands it whatever gateway is currently bound.

---