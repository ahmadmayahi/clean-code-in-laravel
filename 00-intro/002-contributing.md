This book is open to contributions. If you spot an error, have a better example, or want to add a section that fills a gap, you are welcome to submit a pull request.

## What Makes a Good Contribution

Every chapter in this book follows a pattern: explain *why* a practice matters, then show *how* to apply it in Laravel. A good contribution does the same. It does not just show code — it explains the problem the code solves and why the reader should care.

Before you write, read two or three existing chapters to absorb the tone. The writing is direct, opinionated, and practical. No hedging, no filler, no "it depends" without explaining on what. Sentences are short. Paragraphs get to the point. Code examples use real-world scenarios, not `FooBar` abstractions.

Here is what to aim for:

- **Start with the problem.** Show the messy or naive approach first, then introduce the clean version. The reader should feel the pain before seeing the cure.
- **Explain the why, not just the what.** "Use Actions" is not useful. "Use Actions because they give you a single place to test business logic without HTTP overhead" is.
- **Use Laravel 12 and PHP 8.4.** Constructor property promotion, enums, match expressions, typed properties — use modern PHP. No legacy patterns.
- **Keep examples realistic.** Orders, users, subscriptions, payments — things that exist in real applications.
- **Skip the obvious.** Do not explain what a controller is. The reader already knows Laravel. Explain what makes a controller *clean*.

## Using AI

You are free to use AI to help you write. I use it myself. But AI is a drafting tool, not an author. If you paste a chapter into Claude or GPT and submit whatever comes back, it will not match the book's voice — and I will be able to tell.

Here is what works: use AI to generate a first draft, then rewrite it in your own words. Cut the filler it adds. Remove the hedging. Make the sentences shorter. Add your own experience and opinions. The result should sound like a person who has strong views about Laravel, not a language model trying to be helpful.

If you use AI, give it context about the book's style. Here is a prompt that works well:

```
I am contributing to a technical book about clean code in Laravel.
The book's style is:
- Direct and opinionated — no hedging or "it depends" without explaining on what
- Short sentences, short paragraphs
- Every section starts with the problem (messy code), then shows the clean solution
- Code examples use realistic scenarios (orders, users, payments), not FooBar
- Uses Laravel 12 and PHP 8.5 features (constructor property promotion, enums, match, typed properties)
- Explains WHY a pattern matters, not just HOW to use it
- No bold in paragraphs — bold is for list item labels and headings only
- References link to official docs or well-known community articles

Write a section about [YOUR TOPIC]. Start with the problem, show the naive approach,
then present the clean solution with an explanation of why it is better.
Keep it practical — the reader already knows Laravel.
```

Then rewrite the output until it sounds like you, not the model.

## What to Contribute

- **Fix errors.** Typos, broken code examples, outdated patterns, wrong Laravel version behavior.
- **Improve examples.** If a code sample could be clearer or more realistic, improve it.
- **Add missing topics.** If a chapter skips something important, add a section. Open an issue first to discuss scope.
- **Update references.** If a linked article has moved or disappeared, find the new URL or a suitable replacement.

## What Not to Contribute

- Do not rewrite existing sections to match a different opinion. The book is opinionated on purpose. If you disagree with a pattern, open an issue for discussion instead of submitting a rewrite.
- Do not add chapters about topics the book intentionally skips. If it is not here, there is probably a reason. Ask first.
- Do not add comments, docblocks, or type annotations to code you did not change. Keep diffs focused.

## How to Submit

Fork the repository, make your changes, and open a pull request. In the PR description, explain what you changed and why. If you are adding new content, include a brief outline of the section so I can review the direction before the details.
