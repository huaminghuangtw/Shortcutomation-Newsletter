---
title: Cleaner Shortcuts: Stop Writing “Otherwise”
created: 2026-08-23
modified: 2026-08-23
featured: false
tags:
  - dev-tip
---

//todo: another screenshot
While building a shortcut to export my bookmarks (the content I want to enjoy later, books included), I kept tripping over the same ugly pattern. The shortcut was full of `If … Otherwise` blocks, and the `If` branches were often nothing more than a single “Get Value from X” action. All that structure for one little value. That’s when I started thinking about simplifying.

![The cover-fallback shortcut in the Shortcuts editor](https://media.huam.ing/image/cover-fallback.webp "Instead of handling the happy path in its own branch, I only write the fallback — the happy path flows through untouched.")

## The realization: fewer branches, cleaner shortcuts

Most of the time, an `If … Otherwise` in Shortcuts looks like this:

```text
If imageLinks.thumbnail has any value
    Get Value for imageLinks.thumbnail in data
Otherwise
    Get Value for previewLink in data
End If
```

The `If` branch doesn’t really do anything special — it just re-returns the value that was already there. The only branch doing real work is the fallback. So why write two branches when one will do?

Flip the condition, and the fallback becomes the only branch you write:

```text
If imageLinks.thumbnail does not have any value
    Get Value for previewLink in data
End If
```

When `imageLinks.thumbnail` **does** have a value, the whole block is skipped and the value flows straight through to whatever comes next. You only write the fallback — the happy path needs no branch at all.

## A real example: chained book-cover fallback

When I export a book, I want its cover. Google Books usually has one, but not always. So my shortcut tries three sources, deepest fallback last:

1. Get `imageLinks.thumbnail`
2. If thumbnail **doesn’t have any value** → try `previewLink`
3. If previewLink **doesn’t have any value** → build a cover from Open Library’s `cover_i`

Three fallback levels, **zero** `Otherwise` blocks:

```text
If imageLinks.thumbnail does not have any value
    Get Value for previewLink in data

    If previewLink does not have any value
        Get Value for cover_i in data
        Set URL → https://covers.openlibrary.org/b/id/{cover_i}-M.jpg
    End If
End If

Get contents of If Result
```

Each level only handles the case where the previous one failed. Whatever survived gets passed along by `If Result`.

## The principle behind the trick

Before I add an `If … Otherwise`, I now ask myself: **is one of these branches just holding the value that’s already there?** If so, flip it.

- Normal case → value present → you never touch the fallback branch.
- Edge case → value missing → only the fallback runs.

This “guard clause” thinking keeps shortcuts short, easy to scan, and easy to maintain. The book-cover chain reads top-to-bottom as “this, or else this, or else this” — the intent is obvious at a glance.

## How this compares to a normal programming language

If you write code, this pattern should look familiar — it’s a **guard clause**, the same idea as an early `return`:

```python
def get_cover(data):
    if data.get("thumbnail"):                       # happy path first
        return data["thumbnail"]
    if data.get("previewLink"):                     # first fallback
        return data["previewLink"]
    return f"https://covers.openlibrary.org/b/id/{data['cover_i']}-M.jpg"  # last resort
```

Or, more idiomatically, a single line using short-circuit evaluation:

```python
cover = thumbnail or preview_link or f"https://covers.openlibrary.org/b/id/{cover_i}-M.jpg"
```

Normal languages have **expressions** — an `or` that stops at the first real value. Shortcuts has no such operator. The only way to pick between values is the `If` action, so the same logic *must* be written as blocks. The trick — flipping the condition so only the fallback is written — is how you bring that one-line elegance back.

What’s unique to Shortcuts is the **implicit data flow**. In code, every value needs a name and every branch needs an explicit `return`. In Shortcuts, values flow *through* actions, and when an `If` block is skipped, the incoming value just keeps going. There’s no `return` — the value is already on its way. That pass-through is the superpower: it’s why you can delete the happy-path branch entirely and the shortcut still works.

So the guard clause itself isn’t new. But in a normal language you reach for it because the `return` forces you to spell out the flow. In Shortcuts you reach for it because the value flows for you.

## Two caveats

1. **Missing value ≠ empty string.** `doesn’t have any value` is for values that are truly absent (a missing dictionary key, an empty API response). If a value is present but blank (`""`), use `is empty` instead.
2. **Don’t over-nest.** Three fallback levels is about the limit before a shortcut gets hard to follow. If you’re going deeper, consider a Dictionary lookup or a `Repeat` instead.

The next time you reach for `Otherwise`, ask: is this branch earning its place? If it’s just echoing a value, flip the condition and watch the shortcut shrink.
