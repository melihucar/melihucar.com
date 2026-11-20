---
title: "PHP 8.6: Partial Function Application and Pipes"
description: "PHP 8.6 adds partial function application, which makes the pipe operator from PHP 8.5 much easier to use. How it works, a small refactoring, and the other changes in 8.6."
date: 2026-11-20T11:00Z
tags: [PHP, PHP 8.6, Laravel]
---

PHP 8.6 was released on 19 November 2026.[^php] It is a smaller release than 8.0 or 8.1, but it has one feature I expect to use a lot: **partial function application**. On its own it saves a few closures. Together with the pipe operator from PHP 8.5, it makes pipes practical for most code.

Below: how partial application works, what it does for pipes, a small refactoring from my [agent memory series](/articles/giving-a-laravel-ai-agent-a-memory/), and the rest of the release.

## The short version

- Call a function with `?` in place of an argument and you get a closure back that takes that argument. `...` stands for all the remaining arguments.
- The pipe operator `|>` from PHP 8.5 passes a value to a callable that takes **one** argument. Most useful functions take more than one, so in 8.5 that meant wrapping them in arrow functions. In 8.6, `?` does that for you.
- Also new: `clamp()`, `Time\Duration`, default values for readonly properties, a `SortDirection` enum, a native polling API, and the deprecation of `return` inside `finally`.

## Partial application in one example

Here is the example from the PHP manual:[^manual]

```php
function volume(int $length, int $width, int $height): int
{
    return $length * $width * $height;
}

$closure = volume(2, ?, 4);
// The same as: static fn (int $width): int => volume(2, $width, 4)

$closure(3); // 24
```

`?` is exactly one argument. `...` is everything that's left:

```php
$closure = volume(?, 3, ?);
$closure(2, 4); // 24

$closure = volume(2, ...);
$closure(3, 4); // 24
```

Named arguments work too, which is handy for functions whose parameter order you never remember:

```php
$remove = str_replace(subject: ?, search: ?, replace: '');

$remove('a-b-c', '-'); // 'abc'
```

The closure's parameters keep the names, types and by-reference mode of the original function, and it keeps the return type.[^manual] If the arguments and placeholders don't match the function's parameters, you get an `ArgumentCountError` when the closure is *created*, not later when someone calls it.

One difference from an arrow function is easy to miss. The arguments you fill in are evaluated **once**, when the closure is created:

```php
$a = format_report(loadTemplate(), ?);           // loadTemplate() runs once
$b = fn ($data) => format_report(loadTemplate(), $data); // runs on every call
```

Usually that's what you want. If not, use an arrow function.

## Why pipes needed this

PHP 8.5 added the pipe operator. It takes the value on the left and passes it to the callable on the right:

```php
$slug = $title
    |> trim(...)
    |> strtolower(...);
```

That reads nicely for as long as every function takes one argument. As soon as one needs more, you're back to closures, and in 8.5 an arrow function in a pipe also has to be wrapped in parentheses:

```php
// PHP 8.5
$slug = $title
    |> trim(...)
    |> strtolower(...)
    |> (fn (string $s) => preg_replace('/[^a-z0-9]+/', '-', $s))
    |> (fn (string $s) => trim($s, '-'));
```

That's a lot of wrapping for two function calls. With partial application:

```php
// PHP 8.6
$slug = $title
    |> trim(...)
    |> strtolower(...)
    |> preg_replace('/[^a-z0-9]+/', '-', ?)
    |> trim(?, '-');
```

`?` marks where the value goes. This is more useful than it looks, because PHP's standard library is inconsistent about argument order. `preg_replace()` wants the subject last, `trim()` wants it first, `array_filter()` first, `array_map()` second. With a placeholder you don't have to care.

The manual's own example puts it all together:[^manual]

```php
echo ' Hello  World '
    |> trim(...)
    |> str_replace(' ', '', ?)
    |> strtoupper(...)
    |> str_pad(?, 12, '*', STR_PAD_BOTH);

// *HELLOWORLD*
```

## A real refactoring: the keyword recall

In [episode 1 of the agent memory series](/articles/giving-a-laravel-ai-agent-a-memory/), the `recall_memory` tool turned the model's query into keywords like this:

```php
$words = collect(preg_split('/\W+/u', mb_strtolower($request['query'] ?? '')))
    ->filter(fn (string $word) => mb_strlen($word) >= 3)
    ->unique();
```

It works, but you read it inside out: lowercase first, then split, then wrap it in a collection, then filter. And it had a bug, which [Trail](https://github.com/astrophp/trail) made visible: the query "preferred programming language and scripting shell" matched a fact only because both contained the word "and". The filter drops words shorter than three letters, and "and" has three.

Here is the same step in PHP 8.6, with the stop words fixed:

```php
$words = ($request['query'] ?? '')
    |> mb_strtolower(...)
    |> preg_split('/\W+/u', ?)
    |> array_filter(?, fn (string $word) => mb_strlen($word) >= 3)
    |> array_diff(?, ['and', 'the', 'for', 'with', 'that'])
    |> array_unique(...)
    |> array_values(...);
```

It reads top to bottom, in the order things happen, and every step is a plain PHP function. The stop word fix is one extra line in the middle.

> [!NOTE] Pipes don't replace collections
> In a Laravel app I still reach for collections most of the time: they're richer, and the rest of the codebase speaks them. Pipes are nicer with plain strings and arrays, or in code that shouldn't depend on Laravel. Also, episode 2 replaced keyword recall with embeddings, so treat this as an example of the syntax rather than a search improvement.

There's a small performance gain too. The release notes say PHP 8.6 compiles `array_map()` calls with first-class callable or partial application callbacks into loops, so `array_map(strtoupper(...), $items)` no longer pays for a function call per item.[^php]

## The smaller features

### `clamp()`

Keeps a value inside a range:

```php
clamp(-10, 0, 100); // 0
clamp(50, 0, 100);  // 50
clamp(110, 0, 100); // 100
```

It isn't limited to numbers: anything comparable works, including the new `Duration`.[^php] It's handy for any limit that comes from outside, like a `per_page` query parameter:

```php
$perPage = clamp((int) $request->query('per_page', 15), 1, 100);
```

### `Time\Duration`

An immutable value object for a length of time, the kind a stopwatch measures, with nanosecond precision and no time zones involved.[^php] `DateInterval` is about calendar dates; `Duration` is about elapsed time. Durations can be added, multiplied and compared, so `min()` and `clamp()` work on them directly. The release announcement's example is a retry backoff:

```php
use Time\Duration;

$delay = $baseDelay->add($step->multiplyBy(2 ** $attempt));
$delay = min($delay, Duration::fromSeconds(30));

time_nanosleep($delay->seconds, $delay->nanoseconds);
```

### Readonly properties with defaults

Readonly properties can now have a default value; before 8.6 that was a compile-time error. The default counts as initialization, so the constructor can't overwrite it.[^php] This mostly helps with PHP 8.4's interface properties, where a class wants to satisfy a get-only property with a fixed value:

```php
final readonly class SlugRule implements Rule
{
    public string $parser = SlugParser::class;
}
```

### `SortDirection`

A built-in enum with two cases, `SortDirection::Ascending` and `SortDirection::Descending`. Nothing in the standard library requires it yet.[^php] The idea is that libraries share one type instead of each using `'asc'`, `'ASC'`, `SORT_ASC` or `true`.

### Polling many streams at once

The new `Io\Poll` API waits on many streams at once, using the best mechanism each platform has: epoll, kqueue, event ports or WSAPoll.[^php] Most apps won't use it directly; it's mainly for async libraries and long-running servers.

### Also new

- `ReflectionProperty::isReadable()` and `isWritable()`, which take visibility, initialization, property hooks and magic methods into account.
- An `error_include_args` setting that adds function arguments to error messages. It respects `#[\SensitiveParameter]`, but think twice before turning it on anywhere errors are displayed.
- Opt-in exceptions for stream errors, instead of warnings and `false`.
- JSON error messages now say *where* the error is.

## What to watch when upgrading

The deprecation I'm happiest about is `return` inside `finally`. It silently discards exceptions:

```php
function save(): bool
{
    try {
        throw new RuntimeException('Disk full');
    } finally {
        return true; // the exception is gone, and the caller sees success
    }
}
```

In 8.6 this raises a deprecation notice. If you see it in your logs, that code has probably been hiding errors.

A few changes can break things quietly:[^php]

- **Session defaults are stricter**: `session.use_strict_mode` and `session.cookie_httponly` are on, and `session.cookie_samesite` defaults to `Lax`. Laravel manages its own session cookies, but plain PHP apps relying on the old defaults may notice.
- `trim()`, `ltrim()` and `rtrim()` now also strip form feeds (`\f`).
- `fclose()`, `file_put_contents()` and `copy()` return `false` when flushing fails, instead of pretending it worked.
- Many functions now throw `ValueError` for out-of-range values, invalid flags or NUL bytes.

The release also deprecates a long list of old aliases and functions, including `is_double()`, `is_long()`, `is_integer()` and `doubleval()`. The [migration guide](https://www.php.net/manual/en/migration86.php) has the full list.

## Conclusion

PHP 8.6 is a quiet release, but partial application fixes the main problem I had with pipes in 8.5: most functions take more than one argument. With `?`, they fit into a pipe without wrapping. `clamp()`, `Duration` and readonly defaults remove some small helper code as well.

I'm going to try pipes on a few string and array helpers before using them anywhere bigger.

[^php]: [PHP 8.6 Release Announcement](https://www.php.net/releases/8.6/en.php), php.net.
[^manual]: [Partial function application](https://www.php.net/manual/en/functions.partial-application.php), PHP manual.
