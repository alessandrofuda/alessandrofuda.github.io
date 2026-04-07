---
layout: post
title: "Passwordless Auth in Laravel 12: Implementing Magic Link Login with Sanctum"
---

*No passwords, no reset flows, no bcrypt. Just an email, a signed URL, and a Sanctum token. Here's how to implement magic link authentication in Laravel 12 from scratch , including the edge cases that bite you in production.*

Passwords are a liability. Users forget them, reuse them, and your team ends up maintaining reset flows, email verification, and "remember me" cookie logic for years. For LongTermMemory I went fully passwordless from day one: the only way to log in is to receive a magic link by email. This post walks through the complete implementation , backend in Laravel 12 + Sanctum, frontend in React , including the production gotchas that aren't in any tutorial.

---

## The Flow at a Glance

```
1. User submits email → POST /api/auth/magic-link
2. Backend generates a signed URL + a short-lived code → sends email
3. User clicks link → GET /auth/magic-login/{user_id}?signature=...
4. Backend validates signature → generates a one-time code → redirects to frontend
5. Frontend receives code → POST /api/auth/exchange
6. Backend validates code → issues Sanctum token → user is authenticated
```

There are two extra branches: **new users** (who need email verification before getting a login link) and an **OTP fallback** (a 6-digit code in the same email for users whose email client breaks links). Both share the same token-issuing endpoint at the end.

---

## Step 1: Send the Magic Link

The entry point is a single endpoint that accepts an email address:

```php
public function magicLink(Request $request): JsonResponse
{
    $request->validate(['email' => 'required|email']);

    $user = User::where('email', $request->email)->first();

    if ($user && $user->email_verified_at) {
        $url = URL::temporarySignedRoute(
            'magic.login',
            now()->addMinutes(15),
            ['user_id' => $user->id]
        );
        $email = $user->email;
        $status = 200;
    } else {
        $email = $request->email;
        $user = $this->createNewUserInDB($email);
        $url = URL::temporarySignedRoute(
            'magic.register',
            now()->addMinutes(15),
            ['user_id' => $user->id]
        );
        $status = 404;
    }

    $otp = $this->generateAndSaveOtp($user->id);
    Mail::to($email)->send(new MagicLoginLink($url, $otp));

    return response()->json(['status' => $status]);
}
```

A few design decisions here:

**Known vs unknown email.** If the email exists and is verified, the user gets a `magic.login` link. If the email is new or unverified, a user record is created and they get a `magic.register` link (which also marks `email_verified_at` on click). Both flows converge at the same code-generation step.

**`URL::temporarySignedRoute()`** generates a URL with an HMAC signature and an expiry timestamp baked in. Laravel validates both automatically when you call `$request->hasValidSignature()`. The link expires in 15 minutes , long enough to be usable, short enough to limit exposure.

**OTP in the same email.** Every magic link email also contains a 6-digit OTP (`rand(100000, 999999)`), valid for 15 minutes. Users on mobile apps or email clients that mangle URLs can type the code instead. Same security properties, different UX.

**Never enumerate users.** Both branches return HTTP 200 to the caller , the `$status` field inside the JSON body differs (200 vs 404), but the HTTP status code is always 200. This prevents email enumeration via timing or status code differences.

---

## Step 2: Validate the Signature and Generate a Code

When the user clicks the link, the backend validates the signature and exchanges it for a short-lived one-time code:

```php
public function magicLogin(Request $request, $user_id)
{
    if ($errorResponse = $this->validateSignatureOrFail($request)) {
        return $errorResponse;
    }

    $user = User::findOrFail($user_id);
    return $this->generateCodeAndRedirect($user);
}

private function generateCodeAndRedirect(User $user, ?string $redirectTo = null)
{
    $code = Str::uuid()->toString();

    DB::table('magic_login_codes')->insert([
        'code'       => hash('sha256', $code),
        'user_id'    => $user->id,
        'expires_at' => now()->addMinutes(5),
        'used'       => false,
        'created_at' => now(),
    ]);

    $callbackUrl = config('app.frontend_url') . '/auth/callback?code=' . urlencode($code);

    if ($redirectTo !== null) {
        $callbackUrl .= '&redirect_to=' . urlencode($redirectTo);
    }

    return redirect($callbackUrl);
}
```

The signed URL is valid for 15 minutes. After validation, a fresh UUID code is generated , but **only the SHA-256 hash is stored**, never the plaintext. This is the same principle as storing hashed passwords: if your database leaks, raw codes can't be replayed. The code itself lives in the URL for 5 minutes before it expires.

The redirect sends the browser to the React frontend at `/auth/callback?code=<uuid>`, which then exchanges it for a Sanctum token.

---

## Step 3: Exchange the Code for a Token

```php
public function exchangeCodeWithToken(Request $request): JsonResponse
{
    $request->validate(['code' => 'required|string']);

    $record = DB::table('magic_login_codes')
        ->where('code', hash('sha256', $request->code))
        ->where('expires_at', '>', now())
        ->where('used', false)
        ->first();

    if (! $record) {
        return response()->json(['message' => 'Invalid or expired code'], 401);
    }

    DB::table('magic_login_codes')
        ->where('code', hash('sha256', $request->code))
        ->update(['used' => true]);

    $user = User::findOrFail($record->user_id);

    if (! $user->notifications_enabled) {
        $user->update(['notifications_enabled' => true]);
    }

    $token = $user->createToken('auth_token')->plainTextToken;

    return response()->json(['token' => $token]);
}
```

Three checks before issuing a token: the hash matches, the code hasn't expired, and it hasn't been used before. The `used` flag is set to `true` immediately after the record is found, before the token is issued. This stops casual replay attempts , though a fully concurrent double-submit at the exact same millisecond could theoretically pass both `where('used', false)` queries before either update lands. A proper fix wraps the read-and-update in a database transaction; for a low-traffic auth endpoint this race window is acceptable, but worth noting.

The `notifications_enabled` re-enable on login is a deliberate UX choice: users who were auto-disabled after 30 days of inactivity get their reminders back the moment they log in again. Logging in is an implicit signal of renewed interest.

`$user->createToken('auth_token')->plainTextToken` creates a Sanctum personal access token. The frontend stores this in `localStorage` as `auth_token` and sends it as `Authorization: Bearer {token}` on every subsequent request.

---

## The Reverse Proxy Signature Gotcha

In production, the app runs behind Nginx. Signed URLs are generated using `config('app.url')` as the base , which might be `https://api.longtermemory.com`. But the request that arrives at Laravel's validation layer may have `http://localhost` as its host (the proxy doesn't forward `X-Forwarded-Proto` correctly in all configurations).

Laravel's `$request->hasValidSignature()` reconstructs the URL from the incoming request to verify the HMAC. If the scheme or host differs from what was signed, validation silently fails.

The fix is a fallback that normalizes the URL against `config('app.url')` before validating:

```php
private function hasValidAppUrlSignature(Request $request, array $ignoreQuery = []): bool
{
    // Try standard validation first
    $standardMethod = empty($ignoreQuery)
        ? $request->hasValidSignature()
        : $request->hasValidSignatureWhileIgnoring($ignoreQuery);

    if ($standardMethod) {
        return true;
    }

    // Fallback: rebuild the URL using config('app.url') as the base
    $appUrl = rtrim(config('app.url'), '/');
    $normalizedUrl = $appUrl . $request->getRequestUri();
    $normalizedRequest = Request::create($normalizedUrl);

    return URL::hasValidSignature($normalizedRequest, true, $ignoreQuery);
}
```

The test that covers this scenario is worth reading , it sets `APP_URL` to HTTPS, generates a signed URL with that scheme, then sends the request as a relative path (simulating a proxy that strips the scheme):

```php
public function test_magic_login_with_redirect_works_when_scheme_differs_from_app_url(): void
{
    $user = User::factory()->create();

    $httpsUrl = preg_replace('/^http:/', 'https:', config('app.url'));
    config(['app.url' => $httpsUrl]);
    URL::forceRootUrl($httpsUrl);
    URL::forceScheme('https');

    $signedUrl = URL::temporarySignedRoute('magic.login.redirect', now()->addDays(30), ['user_id' => $user->id]);
    $url = $signedUrl . '&redirect_to=' . urlencode('/study-plan/pr/1');

    URL::forceScheme(null); // reset so the test request doesn't force https

    $parsedUrl = parse_url($url);
    $pathAndQuery = ($parsedUrl['path'] ?? '/') . '?' . ($parsedUrl['query'] ?? '');

    $response = $this->get($pathAndQuery);

    $response->assertRedirect();
    $this->assertStringContainsString('/auth/callback', $response->headers->get('Location'));
}
```

Without this fallback, every production login fails silently with a redirect to `/login?error=Invalid+or+expired+link` , a very confusing bug to diagnose.

---

## Open Redirect Protection

The `magic.login.redirect` route accepts a `redirect_to` query parameter so that notification emails can deep-link users directly to their study plan after login. But this parameter must not be part of the URL signature , it's appended after signing because the destination URL is determined at notification send time, not at route generation time.

This means `redirect_to` must be validated separately:

```php
public function magicLoginWithRedirect(Request $request, $user_id)
{
    if (! $this->hasValidAppUrlSignature($request, ['redirect_to'])) {
        return redirect(config('app.frontend_url') . '/login?error=...');
    }

    $user = User::findOrFail($user_id);

    $redirectTo = $request->query('redirect_to');
    if ($redirectTo !== null && !str_starts_with($redirectTo, '/')) {
        $redirectTo = null; // reject any absolute or external URL
    }

    return $this->generateCodeAndRedirect($user, $redirectTo);
}
```

`hasValidAppUrlSignature($request, ['redirect_to'])` is the custom wrapper from the previous section , internally it calls `$request->hasValidSignatureWhileIgnoring(['redirect_to'])`, which validates the HMAC while ignoring that specific query parameter. Then the value itself is checked: only relative paths (starting with `/`) are forwarded. Anything else , `http://evil.com/steal`, `//evil.com`, `javascript:` , is silently dropped.

The test covers this:

```php
public function test_magic_login_with_redirect_ignores_external_redirect_to(): void
{
    // ... generate signed URL
    $url = $signedUrl . '&redirect_to=' . urlencode('http://evil.com/steal');

    $response = $this->get($url);

    $location = $response->headers->get('Location');
    $this->assertStringContainsString('/auth/callback', $location);
    $this->assertStringNotContainsString('redirect_to', $location);
    $this->assertStringNotContainsString('evil.com', $location);
}
```

---

## The React Side: Handling StrictMode's Double Invocation

In React 19 with `<StrictMode>`, effects run **twice in development**. For most effects that's fine. For an auth callback that exchanges a one-time code for a token, it's a problem: the second call hits the endpoint with a code that's already been marked `used`, gets a 401, and the user sees an auth error.

The fix is a ref guard:

```typescript
function AuthCallback() {
  const [searchParams] = useSearchParams();
  const navigate = useNavigate();
  const { completeAuthentication } = usePostAuthFlow();
  const code = searchParams.get('code');
  const redirect_to = searchParams.get('redirect_to');
  const [error, setError] = useState<string | null>(null);
  const hasExchanged = useRef(false);

  useEffect(() => {
    const exchangeCode = async () => {
      if (!code) { navigate('/login'); return; }

      if (hasExchanged.current) return; // prevent StrictMode double-fire
      hasExchanged.current = true;

      try {
        const response = await authApi.exchange(code);

        // Validate redirect_to: relative paths only
        const validRedirect = redirect_to &&
          redirect_to.startsWith('/') &&
          !redirect_to.startsWith('//') &&
          !redirect_to.includes('://') &&
          !['/login', '/auth/callback'].includes(redirect_to)
          ? redirect_to
          : undefined;

        await completeAuthentication(response.token, validRedirect);
      } catch (err: any) {
        setError(err.response?.data?.message || 'Authentication failed');
        setTimeout(() => navigate('/login'), 3000);
      }
    };

    exchangeCode();
  }, [code, navigate, completeAuthentication]);

  // ...
}
```

`useRef` persists across re-renders and across StrictMode's double-mount cycle. Once `hasExchanged.current` is set to `true`, any subsequent invocation of the effect exits immediately. Note that this guard is intentionally not in the dependency array , it's a one-shot flag, not reactive state.

The frontend also validates `redirect_to` independently, even though the backend already validated it. Defense in depth: the frontend ensures it never navigates to an external URL regardless of what arrives in the URL parameter.

After `completeAuthentication()`, the browser auto-detects and sends the user's timezone to `POST /api/user/update-timezone`:

```typescript
// In usePostAuthFlow or AuthCallback, after token is stored
const timezone = Intl.DateTimeFormat().resolvedOptions().timeZone;
await authApi.updateTimezone(timezone); // e.g. "Europe/Rome"
```

This powers the timezone-aware 8 AM study reminder emails , but that's a topic for another post.

---

## Testing: `actingAsUser()` vs `actingAs()`

Laravel's built-in `actingAs($user)` sets the authenticated user but doesn't create a real Sanctum token. This is fine for most tests, but breaks any code that calls `$request->user()->currentAccessToken()` , specifically, the logout endpoint.

The solution is a custom `actingAsUser()` helper in `TestCase`:

```php
// tests/TestCase.php
protected function actingAsUser(int $planId = CommercialPlan::FREE): User
{
    $user = User::factory()->withPlan($planId)->create();
    $token = $user->createToken('auth_token')->plainTextToken;
    $this->withHeader('Authorization', 'Bearer ' . $token);
    return $user;
}
```

This creates a real `personal_access_tokens` record. The logout test can then assert the token was actually deleted:

```php
public function test_logout_deletes_token(): void
{
    $user = $this->actingAsUser();

    $response = $this->postJson('/api/logout');

    $response->assertStatus(200);
    $this->assertDatabaseEmpty('personal_access_tokens');
}
```

With plain `actingAs()`, `currentAccessToken()` returns `null` and the logout controller throws. With the real token helper, both the controller behavior and the database assertion are tested correctly.

---

## What the Database Looks Like

Three tables drive the auth system:

**`magic_login_codes`** , short-lived one-time codes:
```sql
id, code (SHA-256 hash), user_id, expires_at, used (bool), created_at
```

**`otps`** , 6-digit fallback codes:
```sql
id, user_id, otp, expires_at, used (bool), created_at, updated_at
```

**`personal_access_tokens`** , Sanctum tokens (Laravel manages this table automatically):
```sql
id, tokenable_type, tokenable_id, name, token (SHA-256), last_used_at, expires_at, created_at, updated_at
```

Cleanup: the `custom:clean-table-in-db personal_access_tokens` artisan command prunes old tokens on a schedule, keeping the table from growing unbounded.

---

## What I'd Do Differently

**Separate the "new user" and "returning user" email templates.** Currently both get the same `MagicLoginLink` mailable. The register link should have a welcome tone; the login link should be brief. Small thing, but it affects user perception.

**Rate-limit the magic link endpoint.** Right now a bad actor can trigger unlimited emails to any address. A simple `RateLimiter::attempt('magic-link:' . $email, 5, fn() => ..., 60)` per email address per minute would be enough.

**Store the code in Redis instead of MySQL.** The `magic_login_codes` table has high write churn (insert on every login, update on exchange, prune periodically). Redis with a 5-minute TTL is a better fit , auto-expiry, no cleanup job, lower latency.

---

Passwordless auth is one of those features that looks simple until you implement it properly. The signed URL mechanics, the reverse proxy normalization, the open redirect validation, and the StrictMode guard are all edge cases that don't appear in tutorials but will bite you in production. Hopefully this saves you some debugging time.

The full implementation is part of [LongTermMemory](https://longtermemory.com) , an AI-powered study platform built on Laravel 12 and React 19.
