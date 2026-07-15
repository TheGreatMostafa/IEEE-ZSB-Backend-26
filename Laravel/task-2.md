# Laravel Concepts, Explained Differently

## 1. Laravel Gates

**Analogy: A Locksmith Who Only Answers One Question**

Picture a locksmith you keep on retainer. You don't hand him a master key to your whole building, instead, every time someone wants to do something specific, you call him and ask a single yes/no question: "Can this person open the server room?" He checks whatever rule you gave him, answers "yes" or "no," and hangs up. He doesn't keep a file on every employee or track their whole history, he just answers the one question you asked, for the one situation in front of him.

That's a **Gate** in Laravel. It's a closure that answers exactly one authorization question. It isn't attached to a model class, it's a standalone rule you can invoke from anywhere in the app.

---

### Defining a Gate

Gates usually live in the `boot()` method of a service provider (commonly `AppServiceProvider`).

```php
<?php
use Illuminate\Support\Facades\Gate;
use App\Models\Invoice;
use App\Models\User;

public function boot(): void
{
    Gate::define('void-invoice', function (User $user, Invoice $invoice) {
        return $user->id === $invoice->created_by;
    });

    // Not every gate needs a model
    Gate::define('view-billing-dashboard', function (User $user) {
        return $user->department === 'finance';
    });
}
```

The closure's first parameter is always the logged-in user. Anything after that is just extra context you supply when the check runs.

---

### Using a Gate

**In a controller**

```php
<?php
use Illuminate\Support\Facades\Gate;

public function void(Invoice $invoice)
{
    Gate::authorize('void-invoice', $invoice); // 403s automatically on failure

    $invoice->markAsVoided();
}
```

```php
<?php
// Manual branching instead of an automatic 403
if (Gate::allows('void-invoice', $invoice)) {
    // go ahead
}

if (Gate::denies('void-invoice', $invoice)) {
    abort(403);
}
```

**Straight off the user object**

```php
<?php
if ($request->user()->can('void-invoice', $invoice)) {
    // proceed
}
```

**Inside Blade**

```blade
@can('void-invoice', $invoice)
    <button>Void This Invoice</button>
@endcan

@cannot('view-billing-dashboard')
    <p>Ask a finance team member for access.</p>
@endcannot
```

**On a route**

```php
<?php
Route::post('/invoices/{invoice}/void', [InvoiceController::class, 'void'])
    ->middleware('can:void-invoice,invoice');
```

---

### What's Actually Happening Behind the Scenes

**The facade is a thin wrapper**

`Gate::define(...)` doesn't build anything fancy, it forwards to a singleton instance of `Illuminate\Auth\Access\Gate` sitting in the container. `Gate::allows()`, `Gate::authorize()`, all of it, really just `app(Illuminate\Contracts\Auth\Access\Gate::class)` under a friendlier name.

**Rules live in a plain array**

The `Gate` class holds an internal `$abilities` array. Defining a gate is roughly this:

```php
<?php
// Roughly what happens inside Illuminate\Auth\Access\Gate
public function define($ability, $callback)
{
    $this->abilities[$ability] = $callback;
    return $this;
}
```

No hidden registry, no compiled rule engine, just a name pointing at a closure.

**Checking an ability, step by step**

```
Gate::allows('void-invoice', $invoice)
        ↓
Look up $this->abilities['void-invoice']
        ↓
Pull the authenticated user from the active guard
        ↓
Invoke the closure: $callback($user, $invoice)
        ↓
Reduce the result to true/false (or a Response object)
```

If the ability was never defined, or nobody's logged in, the gate defaults to closed, nothing is authorized unless a rule explicitly says so.

**`before()` and `after()` interceptors**

Global override hooks run ahead of any specific ability, the usual place to put "admins bypass everything":

```php
<?php
Gate::before(function (User $user, string $ability) {
    if ($user->hasRole('super-admin')) {
        return true; // short-circuits everything else
    }
});
```

These live in a separate list and are checked first. The moment one returns something other than `null`, that's the final verdict, the ability's own closure never even runs.

**Gates and Policies share one engine**

A `Policy` class (say, `InvoicePolicy` with a `void($user, $invoice)` method) isn't a different mechanism, Laravel quietly registers each policy method as a gate ability, matching model classes to policy classes by naming convention. So `Gate::authorize('void', $invoice)` and `$this->authorize('void', $invoice)` in a controller both run through the identical `Gate` engine described above. Rule of thumb: gates for one-off actions with no natural model home (`view-billing-dashboard`); policies for a cluster of actions that all revolve around one model (`view`, `void`, `refund` on `InvoicePolicy`).

---

## 2. Sanctum vs Passport

**Analogy: A Spare Apartment Key vs. an Airport Boarding Pass System**

Giving a friend a spare key to your apartment is about as simple as trust gets, you hand it over, they can get in, you can ask for it back. No paperwork, no expiration date printed on it, no third parties involved.

An airport works nothing like that. Nobody gets a personal key to the tarmac. Instead, you check in, prove who you are, and get a boarding pass that's valid for one specific flight, one specific gate, within one specific time window, issued by a system built to handle thousands of different travelers, airlines, and connecting flights at once, some of whom are working with partner airlines they've never dealt with directly.

**Sanctum** is the spare key, small, direct, made for your own first-party app talking to your own backend. **Passport** is the airport's boarding-pass infrastructure, a full **OAuth2** server designed for many different clients, scoped access, and often outside developers who need permission to act on a user's behalf.

---

### Side-by-Side

| | **Sanctum** | **Passport** |
|---|---|---|
| **Protocol** | Lightweight tokens / cookie session auth | Complete OAuth2 server |
| **Ideal for** | Your own SPA or mobile app | Platforms with many clients, external integrators |
| **Setup effort** | One migration, one middleware, done | OAuth clients, grant types, scopes to configure |
| **Where tokens live** | A single `personal_access_tokens` table | Dedicated OAuth2 client/token/scope tables |
| **Expiration** | Disabled unless you turn it on | Configurable per grant; short-lived + refresh tokens typical |
| **Permission granularity** | Simple string "abilities" attached to a token | Full OAuth2 scopes |
| **SPA cookie auth** | Built in, first-class support | Overkill, not really its purpose |
| **Third-party developers** | Not really the use case | Exactly what it's designed for |

For the overwhelming majority of apps, anything where only your own frontend talks to your own API, Sanctum is the right default. Passport earns its complexity only once outside parties need delegated, scoped access to user data through a standardized protocol.

---

### What Using Each One Looks Like

**Sanctum: cookie-based SPA auth**

```php
<?php
// routes/api.php
Route::middleware('auth:sanctum')->get('/me', function (Request $request) {
    return $request->user();
});
```

```js
// Frontend: grab a CSRF cookie, then log in normally
await axios.get('/sanctum/csrf-cookie');
await axios.post('/login', { email, password });
```

**Sanctum: issuing a token for a mobile client**

```php
<?php
$token = $user->createToken('ios-app', ['read-orders', 'create-orders'])->plainTextToken;
// Mobile app sends it back as: Authorization: Bearer {token}
```

**Passport: password grant flow**

```php
<?php
Route::post('/oauth/token', [\Laravel\Passport\Http\Controllers\AccessTokenController::class, 'issueToken']);
```

```
Client app sends: client_id, client_secret, username, password
        ↓
Passport's OAuth2 server checks the client credentials and the user's login
        ↓
Responds with a short-lived access_token + a longer-lived refresh_token
        ↓
Client attaches access_token to requests, swaps it for a new one when it expires
```

---

### The Short Version

Building your own app talking to your own API? Start with **Sanctum**. Need outside companies or developers to log in on a user's behalf through a standard, revocable, scoped protocol? That's **Passport**'s entire reason for existing.

---

## 3. XSRF vs CSRF: Different Name, Same Lock

**Analogy: A Notarized Signature**

Imagine a bank that will only release funds from your account if you show up in person and sign in front of a notary, proving that the request to move money is genuinely coming from you, right now, not from someone who merely knows your account number. Without that in-person signature, anyone holding your account number could phone in and drain the account, and the bank would have no way to tell the difference.

**CSRF (Cross-Site Request Forgery)** is the attack that notarized signature defends against. A malicious site quietly gets your browser to fire off a request, "transfer funds," "change my email," "delete my account", to a service you're already logged into, riding on cookies your browser sends automatically. The target server, seeing a valid session cookie, can't tell that request apart from one you actually meant to send.

---

### So Why Two Names?

There's no real technical distinction, `CSRF` and `XSRF` describe the exact same attack and the exact same defense. `CSRF` is the term used in security literature and by most of the industry. `XSRF` is just the abbreviation that stuck in certain frontend tooling (Angular popularized it, and Laravel's cookie name inherited it).

Inside Laravel, both spellings show up, referring to the same protection:

```
CSRF  → the general concept, and the name of the middleware
          (VerifyCsrfToken) plus the @csrf Blade directive

XSRF  → specifically the XSRF-TOKEN cookie Laravel sets,
          read automatically by JS HTTP clients like Axios
          and echoed back as a request header
```

---

### How Laravel Wires This Up

**Server-rendered forms: the `@csrf` directive**

```blade
<form method="POST" action="/orders">
    @csrf
    <input type="text" name="product_id">
    <button type="submit">Place Order</button>
</form>
```

```blade
{{-- Compiles to a hidden field --}}
<input type="hidden" name="_token" value="aZ7q1...">
```

The `VerifyCsrfToken` middleware confirms this `_token` matches the value stored in the user's session before letting the request through.

**JavaScript frontends: the `XSRF-TOKEN` cookie**

For SPAs, Laravel also drops an `XSRF-TOKEN` cookie on every response. Axios (and similar libraries) reads that cookie by default and automatically re-sends it as an `X-XSRF-TOKEN` header, no manual wiring needed in most setups.

```
Response includes cookie:   XSRF-TOKEN=aZ7q1...
        ↓
Axios reads it automatically
        ↓
Axios attaches header:      X-XSRF-TOKEN: aZ7q1...
        ↓
VerifyCsrfToken checks it against the same session token
```

Two delivery mechanisms, one underlying check, a hidden form field for Blade views, a cookie/header pair for JS clients, both validated by the same middleware against the same session-stored value.

---

## 4. Eloquent Relationships

**Analogy: A Company Org Chart**

An org chart doesn't get redrawn every time someone asks "who reports to this manager?", the reporting lines are defined once, and from then on you can trace them in either direction: manager down to reports, or employee up to their manager. Eloquent relationships work the same way, define the connection between two models once, then walk between them like ordinary properties, no manual `JOIN` required.

---

### The Main Relationship Types

**`hasOne`**: one row owns exactly one related row.

```php
<?php
class Employee extends Model
{
    public function badge()
    {
        return $this->hasOne(AccessBadge::class);
    }
}
// Usage: $employee->badge
```

**`hasMany`**: one row owns a set of related rows.

```php
<?php
class Employee extends Model
{
    public function timesheets()
    {
        return $this->hasMany(Timesheet::class);
    }
}
// Usage: $employee->timesheets (a collection)
```

**`belongsTo`**: the reverse direction; this row is owned by a parent.

```php
<?php
class Timesheet extends Model
{
    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }
}
// Usage: $timesheet->employee
```

**`belongsToMany`**: a many-to-many link through a pivot table.

```php
<?php
class Employee extends Model
{
    public function projects()
    {
        return $this->belongsToMany(Project::class); // default pivot: employee_project
    }
}
// Usage: $employee->projects, $employee->projects()->attach($projectId)
```

**`hasManyThrough`**: reach a far-away relation through a middle model.

```php
<?php
class Department extends Model
{
    public function timesheets()
    {
        // Department -> Employees -> Timesheets
        return $this->hasManyThrough(Timesheet::class, Employee::class);
    }
}
```

---

### Why It's Worth Setting Up Properly

Once defined, Eloquent handles the SQL for you every single time:

```php
<?php
$employee = Employee::find(12);

$employee->timesheets;   // SELECT * FROM timesheets WHERE employee_id = 12
$employee->badge;        // SELECT * FROM access_badges WHERE employee_id = 12
$employee->projects;     // joins through the pivot table automatically
```

Relationships can be **eager loaded** with `with()` to sidestep the N+1 query trap, one extra query total instead of one query per row:

```php
<?php
// N+1 trap: 1 query for employees, then 1 more PER employee
$employees = Employee::all();
foreach ($employees as $employee) {
    echo $employee->timesheets->count();
}

// Fixed: 2 queries total, regardless of how many employees there are
$employees = Employee::with('timesheets')->get();
```

Bottom line: describe the connection between your models once, and Eloquent generates the correct query every time you follow that connection afterward.
