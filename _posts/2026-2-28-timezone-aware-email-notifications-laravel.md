---
layout: post
title: "Timezone-Aware Email Notifications in Laravel: Sending at 8 AM in Every User's Local Time"
---

*The problem sounds simple: send a study reminder email at 8 AM. The catch is that your users live in Tokyo, Rome, New York, and Nairobi. Here's how to build a Laravel artisan command that fires for every user at their local 8 AM , including the N+1-avoidance pattern, the deduplication scheme, and the rate-limit stagger that keeps the email provider happy.*

---

## Why "Send at 8 AM" Is Non-Trivial

A cron job that runs at `0 8 * * *` sends email at 8 AM UTC , which is fine for users in London in winter and confusing for everyone else. The standard alternative, running the job every hour and checking whether it's currently 8 AM in each user's timezone, introduces its own problems: N+1 queries, duplicate sends when the cron overlaps, and edge cases around NULL timezone values.

[LongTermMemory](https://longtermemory.com) sends daily study reminder emails to users who have due flashcard items. The requirement: each notification lands at 8 AM in the user's local time, contains direct links to their study sessions (via magic link deep-links), and fires at most once per day regardless of cron retries.

The implementation is a single artisan command, `custom:send-study-review-notifications`, that runs hourly.

---

## The Core Idea: Collect Timezones at Target Hour

Rather than querying users first and then checking their timezones, the command inverts the approach: it starts by collecting every IANA timezone identifier where the current local hour matches the target, then queries only the users in those timezones.

```php
protected $signature = 'custom:send-study-review-notifications {--hour=8 : The local hour to target (0-23)}';

private function getCandidateUserIds(): Collection
{
    $targetHour = (int) $this->option('hour');
    $targetTimezones = collect(timezone_identifiers_list())
        ->filter(fn (string $tz) => Carbon::now($tz)->hour === $targetHour)
        ->values();

    if ($targetTimezones->isEmpty()) {
        $this->info("No timezones currently at {$targetHour}:00.");
        return collect();
    }
    // ...
}
```

`timezone_identifiers_list()` returns all ~400 valid IANA timezone identifiers. `Carbon::now($tz)->hour` gives the current local hour for each one. At any given moment, roughly 15,25 of those timezones will be at hour 8, depending on DST state.

This is evaluated in PHP, not SQL , a `collect()->filter()` loop over 400 strings is fast enough (microseconds) and avoids the complexity of storing UTC-offset data in MySQL.

---

## The Two-Query Pattern: Candidates, Then Due Items

Fetching users and their due items in a single query would require a complex self-join that's hard to read and harder to extend. The command uses two separate queries:

**Query 1 , candidate users:** Who is at 8 AM right now and has at least one study plan?

```php
$query = DB::table('users')
    ->where('notifications_enabled', true)
    ->whereIn('timezone', $targetTimezones)
    ->whereExists(fn ($q) => $q->select(DB::raw(1))
        ->from('projects')
        ->whereColumn('projects.user_id', 'users.id')
        ->whereExists(fn ($q2) => $q2->select(DB::raw(1))
            ->from('study_plans')
            ->whereColumn('study_plans.project_id', 'projects.id')
        )
    );
```

`DB::raw(1)` in the `SELECT` of the EXISTS subquery is idiomatic SQL: the optimizer ignores the selected value in an EXISTS context, so `SELECT 1` signals "I only care whether a row exists." This is a lint-friendly convention, not a performance trick.

**NULL timezone handling.** Users who registered before timezone detection was added have `NULL` in the `timezone` column. The command treats them as UTC:

```php
if ($targetTimezones->contains('UTC')) {
    $query->orWhere(fn ($q) => $q->where('notifications_enabled', true)
        ->whereNull('timezone')
        ->whereExists(fn ($q2) => $q2->select(DB::raw(1))
            ->from('projects')
            ->whereColumn('projects.user_id', 'users.id')
            ->whereExists(fn ($q3) => $q3->select(DB::raw(1))
                ->from('study_plans')
                ->whereColumn('study_plans.project_id', 'projects.id')
            )
        )
    );
}
```

The condition only activates when UTC is among the target timezones , at any other hour, NULL-timezone users are simply excluded.

---

**Query 2 , due items, grouped by user:** Which projects actually have items to review?

```php
private function getDueProjectsByUser(Collection $candidateUserIds): Collection
{
    return DB::table('study_plans')
        ->join('projects', 'study_plans.project_id', '=', 'projects.id')
        ->whereIn('projects.user_id', $candidateUserIds)
        ->where(fn ($q) => $q->where(fn ($q2) => $q2
                ->where('study_plans.scheduled_at', '<=', Carbon::now())
                ->where('study_plans.is_strict', false)
            )->orWhereNull('study_plans.scheduled_at')
        )
        ->select('projects.user_id', 'study_plans.project_id')
        ->distinct()
        ->get()
        ->groupBy('user_id');
}
```

The query returns one row per `(user_id, project_id)` pair. `->groupBy('user_id')` is a Collection method (not SQL `GROUP BY`) that organizes those rows into a keyed structure:

```
[
    5  => [ { user_id: 5, project_id: 10 }, { user_id: 5, project_id: 23 } ],
    12 => [ { user_id: 12, project_id: 31 } ],
]
```

The due item filter mirrors the session fetching logic: an item qualifies if its `scheduled_at <= now()` and `is_strict = false`, or if `scheduled_at IS NULL` (new item, never reviewed). Strict items , those rated `again` or `hard`, meaning the algorithm wants the user to revisit them soon , are excluded from the notification trigger. They'll reappear once their countdown elapses.

Two queries. No N+1. No model hydration on the candidate pass (just `pluck('id')`).

---

## Deduplication with `insertOrIgnore`

The cron runs hourly. At 8:00 AM UTC, the command fires. If it also runs at 8:05 due to a retry or overlap, the same users would get a second email. The `notification_logs` table prevents this:

```php
$userToday = Carbon::now($user->timezone ?? 'UTC')->toDateString();
$projectIds = $userProjects->pluck('project_id')->values();

$inserted = DB::table('notification_logs')->insertOrIgnore([
    'user_id'    => $user->id,
    'type'       => 'study_review_reminder',
    'sent_date'  => $userToday,
    'created_at' => now(),
    'updated_at' => now(),
]);

if ($inserted === 1) {
    $user->notify((new StudyReviewReminder($projectIds))->delay(...));
}
```

`notification_logs` has a unique composite index on `(user_id, type, sent_date)`. `insertOrIgnore` maps to `INSERT IGNORE` in MySQL , if a row with that combination already exists, the insert silently does nothing and returns 0. If it succeeds, it returns 1 and the notification is dispatched.

Crucially, the `sent_date` is the user's local date (`Carbon::now($user->timezone ?? 'UTC')->toDateString()`), not UTC. A user in UTC+14 (Line Islands) whose 8 AM fires at `2026-03-02 18:00 UTC` gets `sent_date = 2026-03-03` , their local date , so a retry at 18:05 UTC still deduplicates correctly.

---

## Rate Limiting: 1-Second Stagger

Notifications are dispatched via Laravel's queue. The underlying mail provider ([Resend](https://resend.com), on the free plan) has a 2 req/s rate limit. Queuing all notifications simultaneously would burst well past that.

The fix is incremental dispatch delay:

```php
$user->notify(
    (new StudyReviewReminder($projectIds))->delay(now()->addSeconds($sentCount))
);
$sentCount++;
```

- User 1 → 0s delay (immediate)
- User 2 → 1s delay
- User 3 → 2s delay
- …

The delay is set at dispatch time, before the notification hits the queue. Each notification is processed at least 1 second after the previous one, keeping throughput at ≤ 1 req/s , safely under the 2 req/s ceiling. The comment in the code flags this as a free-plan constraint: on a paid plan with higher rate limits, the stagger can be reduced or removed.

---

## The Notification: Deep-Link Magic and Signed Unsubscribe URLs

The `StudyReviewReminder` notification is queued (`implements ShouldQueue`) and builds one action URL per project:

```php
foreach ($this->projectIds as $projectId) {
    $projectName = Project::find($projectId)?->name ?? "Project #{$projectId}";
    $signedUrl = URL::temporarySignedRoute(
        'magic.login.redirect',
        now()->addDays(30),
        ['user_id' => $notifiable->id]
    );
    // redirect_to is appended after signing to avoid %2F double-encoding issues.
    $actionUrl = $signedUrl . '&redirect_to=' . urlencode("/study-plan/pr/{$projectId}");

    $projects[] = ['name' => $projectName, 'url' => $actionUrl];
}
```

Each link is a 30-day temporary signed URL for `magic.login.redirect` , the same passwordless login route used elsewhere in the app. After validating the signature, the backend generates a short-lived one-time code, then redirects the browser to `/auth/callback?code=...&redirect_to=/study-plan/pr/{projectId}`. The user lands directly in their study session, authenticated, without entering any credentials.

`redirect_to` is appended after signing rather than included in the signed payload because the destination URL is determined at notification send time, and appending a `%2F`-encoded path to an already-signed URL would mangle the HMAC. The backend validates `redirect_to` separately, accepting only relative paths that start with `/`.

**The unsubscribe URL** uses a permanent signed route (no expiry):

```php
$unsubscribeUrl = URL::signedRoute(
    'notifications.unsubscribe',
    ['user_id' => $notifiable->id]
);
```

`URL::signedRoute()` (without `temporary`) generates a URL that's valid indefinitely. When the user clicks it, the `notifications.unsubscribe` route validates the signature and sets `notifications_enabled = false` on the user record. No login required, no expiry to worry about. The user simply never gets another reminder.

The auto-re-enable on next login: when the user authenticates again (via magic link), the `exchangeCodeWithToken` endpoint checks `notifications_enabled` and flips it back to `true` if it was disabled. Logging in is treated as an implicit signal of renewed interest.

---

## Testing With Frozen Time

Timezone logic is notoriously hard to test without time control. `Carbon::setTestNow()` makes it tractable:

```php
protected function tearDown(): void
{
    Carbon::setTestNow(); // reset after each test
    parent::tearDown();
}

public function test_sends_notification_to_user_in_8am_timezone(): void
{
    Notification::fake();

    // Freeze time so UTC is 08:00
    Carbon::setTestNow(Carbon::create(2026, 3, 3, 8, 0, 0, 'UTC'));

    $user = $this->createUserAtEightAm('UTC');
    $project = Project::factory()->create(['user_id' => $user->id]);
    StudyPlan::factory()->due()->create(['project_id' => $project->id]);

    $this->artisan('custom:send-study-review-notifications');

    Notification::assertSentTo($user, StudyReviewReminder::class);
}
```

The test suite covers seven cases:
- User at 8 AM in their timezone → notification sent
- User outside the 8 AM window → nothing sent
- User with `notifications_enabled = false` → nothing sent
- Existing `notification_logs` row for today → `insertOrIgnore` skips the send
- Only strict study plans (e.g., rated `again` 5 minutes ago) → nothing sent
- All items scheduled in the future → nothing sent
- New items with `scheduled_at IS NULL` → notification sent

The `due()` and `future()` factory states from `StudyPlanFactory` keep the fixtures readable:

```php
public function due(): static
{
    return $this->state(['scheduled_at' => now()->subHour()]);
}

public function future(): static
{
    return $this->state(['scheduled_at' => now()->addDay()]);
}
```

---

## What I'd Do Differently

**Batch the `Project::find()` calls inside the notification.** `StudyReviewReminder::toMail()` calls `Project::find($projectId)` in a loop , one query per project. For a user with ten projects, that's ten queries inside a queued job. A single `Project::whereIn('id', $this->projectIds)->get()->keyBy('id')` before the loop would reduce it to one.

**Add a configurable notification window.** The `--hour` option already makes the target hour configurable, but there's no way to send a *second* notification (e.g., an evening reminder at 20:00) without running the command with `--hour=20` and managing two cron entries. A window-based approach , "send between 7 and 9 AM, once per day" , would be more resilient to users whose 8 AM falls between two hourly runs.

**Prune `notification_logs` on a schedule.** The table grows by one row per user per day. A weekly cleanup (`DELETE WHERE sent_date < NOW() - INTERVAL 7 DAY`) prevents it from becoming a performance liability as the user base scales.

---

Timezone-aware notifications look like a three-liner until you account for NULL timezones, N+1 queries, deduplication across cron retries, and email provider rate limits. The implementation pattern , collect target timezones in PHP, query users with EXISTS, dedup with `insertOrIgnore`, stagger dispatch , handles all four without anything exotic.

