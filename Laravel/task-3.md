# Laravel Concepts, Explained From Scratch

## 1. The N+1 Query Problem

**Analogy: A Bee That Flies Back to the Hive After Every Single Flower**

A smart bee visits a whole patch of flowers in one outing, collects pollen from each one, and flies home a single time with everything it gathered. An exhausted, inefficient bee would instead visit one flower, fly all the way back to the hive to drop off that one bit of pollen, then fly back out to the *next* flower, and repeat, covering the same ground over and over for work that one trip could have handled.

The **N+1 query problem** is your application acting like that exhausted bee. You run **1** query to fetch a batch of records, then **N** more queries, one per record, to fetch something related to each of them. Load 200 orders and then ask each order for its packages one at a time, and you've just made 201 trips to the database when 2 would have covered it.

---

### Watching It Happen

```php
<?php
$orders = Order::all(); // Trip #1: SELECT * FROM orders

foreach ($orders as $order) {
    echo $order->packages->count(); // Trip #2, #3, #4... one per order!
}
```

Nothing here looks off, `$order->packages` reads like you're just checking a value on an object. But Eloquent relationships are **lazy** by default: the first time `packages` is touched on a given order, Eloquent fires off a brand-new `SELECT * FROM packages WHERE order_id = ?` right then and there. Do that inside a loop over 200 orders, and you get 200 separate, individually-planned database round trips, even though the database is perfectly capable of returning all of that data in one shot.

---

### The Fix: Eager Loading

```php
<?php
$orders = Order::with('packages')->get();
// Trip #1: SELECT * FROM orders
// Trip #2: SELECT * FROM packages WHERE order_id IN (1, 2, 3, ...200)

foreach ($orders as $order) {
    echo $order->packages->count(); // Already sitting in memory, no new trips
}
```

`with('packages')` tells Eloquent up front: "every order in this result is going to need its packages, so grab them all now, together." Instead of the hive-and-back routine, Eloquent makes one extra trip using a single `WHERE order_id IN (...)` clause, then sorts the results back onto their matching orders entirely in PHP. Two trips total, whether you have 5 orders or 50,000.

Grab several relationships, even nested ones, in the same pass:

```php
<?php
$orders = Order::with(['packages', 'customer', 'packages.trackingEvents'])->get();
```

---

### Making Laravel Yell at You Before It Becomes a Problem

You can force Laravel to throw an error the instant a lazy load happens somewhere you didn't plan for it, much better to catch it on your own machine than to notice a slow endpoint weeks later in production.

```php
<?php
// AppServiceProvider::boot()
use Illuminate\Database\Eloquent\Model;

Model::preventLazyLoading(! app()->isProduction());
```

With this flipped on, touching an un-eager-loaded relationship in local development throws a `LazyLoadingViolationException` right away, pointing straight at the missing `with()`, instead of a production server quietly absorbing hundreds of avoidable queries per request without complaint.

---

### Digging Into Why the Fix Works

**1. Lazy loading is a magic getter, not a real property**

Writing `$order->packages` doesn't touch a real class attribute, `packages` isn't declared anywhere on the model. Eloquent leans on PHP's `__get()` magic method: it checks whether `packages` matches a defined relationship method, checks whether that relationship has already been loaded into an internal `$relations` array, and if not, calls `packages()`, runs its query immediately, and stashes the result. That "run it the instant it's touched" behavior is exactly the mechanism that snowballs into N+1 inside a loop.

**2. Eager loading just fills the cache ahead of time**

`with('packages')` runs the relationship query *before* your loop ever starts, and stores every result into each order's `$relations` array under the `packages` key. By the time your `foreach` reaches `$order->packages`, `__get()` checks that cache first, finds it already full, and hands it back, no query fires. It isn't smarter querying so much as the data already being there, waiting.

**3. One query, matched up in memory afterward**

Internally, eager loading collects every parent's key (`[1, 2, 3, ...200]`), runs one `WHERE order_id IN (...)` query against the related table, and then loops through those results in PHP, bucketing each one back onto its correct parent order using a hash map keyed by the foreign key. That's the whole trick behind turning 201 queries into 2, the matching work moves from the database (many slow round trips) into PHP (one fast in-memory pass).

**4. Counting doesn't need to load anything**

If all you actually want is a number, like `$order->packages->count()` above, pulling every full package row into memory just to count them is wasted work. `withCount()` skips that:

```php
<?php
$orders = Order::withCount('packages')->get();

foreach ($orders as $order) {
    echo $order->packages_count; // A single integer, no package rows loaded at all
}
```

This runs as a `COUNT(*)` subquery attached to the main query, so you get the number without ever hydrating the related models.

---

## 2. Attach, Sync, and Detach on Many-to-Many Relationships

**Analogy: Curating a Playlist**

Adding one more song to a playlist you're building is a small, additive move, nothing else on the playlist changes. Handing someone a finished tracklist and saying "make the playlist match this exactly, add whatever's missing and drop whatever isn't on here" is a different kind of instruction entirely. And pulling one specific song off the playlist by name, without disturbing anything else, is a third, separate action again.

Those three moves map almost exactly onto how Laravel handles a **`belongsToMany`** relationship through its pivot table: **attach** (add), **sync** (make the whole set match exactly), and **detach** (remove), no raw pivot-table SQL required.

---

### The Setup

```php
<?php
class Playlist extends Model
{
    public function songs()
    {
        return $this->belongsToMany(Song::class); // uses a playlist_song pivot table by default
    }
}
```

Everything below quietly operates on that hidden `playlist_song` table for you.

---

### `attach()`: Add, Touch Nothing Else

```php
<?php
$playlist = Playlist::find(1);

$playlist->songs()->attach($songId);          // one song
$playlist->songs()->attach([4, 7, 12]);        // several at once

// with extra pivot data, e.g. who added it
$playlist->songs()->attach($songId, ['added_by' => auth()->id()]);
```

`attach()` only inserts, it never removes an existing pairing. Call it twice with the same song and, absent a unique constraint on the pivot table, you can end up with a **duplicate** row, so it's meant for cases where you genuinely know you're adding something new.

---

### `detach()`: Remove, Touch Nothing Else

```php
<?php
$playlist->songs()->detach($songId);   // pull one song off
$playlist->songs()->detach([4, 7]);    // pull several off
$playlist->songs()->detach();          // clear the whole playlist
```

`detach()` deletes matching pivot rows and stops there, the songs themselves keep existing, and any *other* playlist containing those same songs is completely unaffected. Called with no arguments, it empties this one playlist while every other playlist's pairings sit untouched.

---

### `sync()`: Make It Match, Exactly

```php
<?php
$playlist->songs()->sync([4, 7, 12]);
```

`sync()` is the "here's the finished tracklist" move: after this call, the playlist contains **exactly** songs 4, 7, and 12, nothing more, nothing less. Laravel compares that array against what's currently attached and does the smallest amount of work needed to close the gap:

```
Currently on the playlist: [4, 9, 12]
sync([4, 7, 12]) called
        ↓
Song 9 isn't in the new list   → removed
Song 7 is new                  → added
Songs 4 and 12 are in both     → left completely alone
```

That makes `sync()` a natural match for something like a "check the songs you want on this playlist" form, you don't manually diff anything yourself, you just submit the final desired state and let Laravel reconcile it.

**Two close relatives:**

```php
<?php
// syncWithoutDetaching: only ever adds, never removes (attach, but safely deduplicated)
$playlist->songs()->syncWithoutDetaching([4, 7]);

// toggle: flips each one: adds it if absent, removes it if present
$playlist->songs()->toggle([4, 7]);
```

---

### Digging Into the Pivot Table Mechanics

**1. No special engine, just insert/delete calls on one plain table**

`attach()`, `detach()`, and `sync()` are all thin wrappers around ordinary `insert()` and `delete()` calls against the pivot table, using foreign keys Laravel already knows about (`playlist_id` and `song_id` by default, or whatever you specify explicitly via `belongsToMany(Song::class, 'playlist_song', 'playlist_id', 'song_id')`).

```php
<?php
// Roughly what attach() does internally
public function attach($id, array $attributes = [])
{
    $this->newPivotStatement()->insert([
        $this->foreignPivotKey => $this->parent->getKey(),
        $this->relatedPivotKey => $id,
        ...$attributes,
    ]);
}
```

**2. `sync()` diffs before it touches anything**

`sync()` doesn't blindly wipe the pivot rows and reinsert everything, that would be wasteful and would needlessly reset any pivot `created_at` timestamp or auto-incrementing pivot ID. Instead, it first `SELECT`s the currently attached IDs, computes the difference in PHP (which ones to drop, which ones to add), and only runs `insert`/`delete` for rows that actually changed. It even hands back a report of exactly what happened:

```
sync() returns:
[
    'attached' => [7],
    'detached' => [9],
    'updated'  => [],
]
```

That return value is genuinely handy, you could, for instance, only fire a notification for the newly attached IDs.

**3. Why that matters for timestamps**

Because `sync()` only touches rows that actually changed, a song attached to the playlist last month and still present today keeps its original pivot `created_at`, it's never deleted and reinserted purely because it happened to appear on both the old and new lists. That's also why calling `sync()` twice in a row with the same array is a safe no-op the second time, a naive "delete everything, then reinsert everything" approach wouldn't have that property.

---

## 3. What Livewire Actually Is

**Analogy: A Pen Pal With No Short-Term Memory**

Imagine writing to a pen pal who has no memory of your last letter by the time the next one arrives, except every envelope you send includes a little summary card recapping exactly where the two of you left off. They read the card, pick the conversation back up seamlessly, write their reply, and send back an updated summary card of their own for next time. From the outside, it looks like a continuous conversation. In reality, neither side is "remembering" anything between letters, the state is being handed back and forth, written down, every single round trip.

**Livewire** builds interactive Laravel pages using exactly that trick. You write ordinary PHP and Blade; between each interaction, your component's data gets packaged up, sent to the browser, and handed right back on the next request, so the page feels alive and stateful, even though PHP itself has no memory between HTTP requests.

---

### A Minimal Livewire Component

```php
<?php
// app/Livewire/CartQuantity.php
namespace App\Livewire;

use Livewire\Component;

class CartQuantity extends Component
{
    public int $quantity = 1;

    public function increase()
    {
        $this->quantity++;
    }

    public function decrease()
    {
        $this->quantity = max(1, $this->quantity - 1);
    }

    public function render()
    {
        return view('livewire.cart-quantity');
    }
}
```

```blade
{{-- resources/views/livewire/cart-quantity.blade.php --}}
<div>
    <button wire:click="decrease">-</button>
    <span>{{ $quantity }}</span>
    <button wire:click="increase">+</button>
</div>
```

```blade
{{-- Dropped into any normal Blade page --}}
<livewire:cart-quantity />
```

Tapping either button updates `$quantity` and re-renders the number, live, with no full page reload, yet every bit of logic here is plain server-side PHP. No hand-built API endpoint, no JavaScript state management, and (for a component this simple) no build step to get it working.

---

### What Happens the Moment You Click "+"

```
User clicks <button wire:click="increase">
        ↓
Livewire's JS runtime catches the click (page does not reload)
        ↓
It sends the current component's "summary card" (serialized state) back to the server,
along with "please run increase() on this instance"
        ↓
Laravel rebuilds a CartQuantity instance from that summary, with $quantity restored
        ↓
increase() runs on the server; $quantity goes from 1 to 2
        ↓
render() runs again, producing fresh HTML
        ↓
Livewire compares old HTML to new HTML
        ↓
Only the changed part of the DOM (the number) gets swapped in the browser
```

The "memory" of `$quantity` genuinely isn't sitting in a live PHP process the whole time, it's being packed into that summary card and unpacked fresh on every request, which is exactly why `$this->quantity++` can behave like an ordinary, persistent object property even though HTTP itself forgets everything between requests.

---

### Where Livewire Tends to Show Up

| Feature | What You'd Otherwise Hand-Build |
|---|---|
| Search boxes that filter results as you type | JS fetch calls + a dedicated search endpoint |
| Forms with instant validation feedback | A JS validation library duplicating server-side rules |
| Paginated tables with no full reload | A JS pagination widget + API calls |
| Dashboards that feel "live" | A frontend framework (Vue/React) plus state management |
| Multi-step wizards with file uploads | A dedicated JS upload library and custom step logic |

---

### Digging Into Why It Feels Like Magic

**1. `wire:click` is a plain JS event listener, attached at runtime**

When the page loads, Livewire's small JavaScript bundle scans the DOM for `wire:*` attributes (`wire:click`, `wire:model`, `wire:submit`, and so on) and wires up ordinary event listeners on those elements. `wire:click="increase"` means nothing more mysterious than: "on click, send a request telling the server to call `increase()` on this exact component instance." There's no custom template compiler here, it's standard Blade plus a thin layer of attributes JavaScript knows how to read.

**2. State is a serialized snapshot, not a lingering PHP process**

PHP processes usually don't survive between HTTP requests, so Livewire can't literally leave your `CartQuantity` object sitting in server memory waiting for the next click. After every render, it serializes the component's public properties into a snapshot (embedded right in the page, alongside a checksum). When the next request lands, Livewire deserializes that snapshot back into a fresh `CartQuantity` instance with `$quantity` restored, runs whichever method was requested, and packs a new snapshot for the next round.

**3. Only public properties get carried along**

Only **public** properties, like `public int $quantity`, are tracked through this cycle, Livewire uses reflection on the component class to know exactly what to serialize and restore. Private or protected properties, and any ordinary local variables inside a method, don't survive between requests, since Livewire has no way of knowing it should preserve them.

**4. `wire:model` rides the exact same round trip**

```blade
<input type="text" wire:model="search">
```

This isn't a separate mechanism from `wire:click`, typing triggers a debounced event that sends the new value to the server, sets `$this->search`, and (depending on how it's configured) can trigger a re-render immediately, going through the identical "snapshot → request → diff → patch the DOM" pipeline described above. It's also why hammering `wire:model` on every keystroke can generate a lot of network traffic, Livewire debounces by default, but the underlying mechanism is still a genuine round trip to the server for each update, not something resolved purely in the browser.

**5. The trade-off against a traditional SPA**

Livewire deliberately keeps rendering logic on the server instead of exposing a JSON API for a JavaScript framework to consume. In exchange for writing far less JavaScript and reusing your existing PHP/Blade and validation logic, you accept a network round trip to the server for interactions a client-heavy SPA might otherwise resolve entirely in the browser. For most admin panels, dashboards, and form-heavy CRUD apps, that trade favors Livewire's simplicity. For offline-capable, animation-heavy, or extremely latency-sensitive interfaces, a dedicated frontend framework may still make more sense.

---

### The Short Version

Livewire builds interactive-feeling pages out of plain PHP and Blade by treating your component's public state like a summary card, re-sent and rebuilt on every interaction, swapping only the pieces of the DOM that actually changed, giving you a good chunk of a JavaScript framework's interactivity without ever leaving the Laravel ecosystem.
