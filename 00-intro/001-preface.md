Laravel makes it easy to build things. Maybe too easy. You scaffold a project, generate some models, write a few controllers, and you have a working application in an afternoon. The problem shows up three months later when you need to change something and realize that nobody — including you — can follow the code.

This is not a Laravel problem. It is a habits problem. The framework gives you powerful tools and sensible defaults, but it does not force you to use them well. You can write beautiful, maintainable Laravel code or you can write a mess that happens to work. The framework does not care either way.

This book is about choosing the first option.

I wrote it because I spent years learning these lessons the hard way — through messy codebases, painful refactors, and code reviews that should have been shorter. Every chapter addresses a real problem I have seen in production Laravel applications, and every solution is something I actually use.

## Who this book is for

You already know Laravel. You can build applications with it. What you want is to build them *better* — code that your teammates can read, that you can maintain six months from now, and that does not collapse under its own weight as the project grows.

If you are new to Laravel, start with the [official documentation](https://laravel.com/docs). Come back here once you have built a project or two.

## Pragmatism Over Dogma

This book presents patterns, not commandments. Every pattern in here — Actions, DTOs, Form Requests, thin controllers — exists to solve a real problem at a certain scale. If you do not have the problem yet, you do not need the pattern yet.

A solo developer building a five-route CRUD application does not need Actions, DTOs, and dedicated Form Requests for every endpoint. A controller with five lines of validation and a simple `create()` call is perfectly fine. It is readable, it is obvious, and extracting it into three separate files would add ceremony without clarity. That is not clean code. That is noise.

There is a name for the opposite of spaghetti code: **ravioli code** — dozens of tiny classes that each do almost nothing, forcing you to click through twelve files to understand a single operation. This book does not advocate for that. If you find yourself creating a `GetFlightsAction` that contains nothing but `Flight::active()->get()`, you have gone too far. A model with a well-named scope, called directly from a controller, is already clean. Not everything needs to be wrapped in another class.

### But Teams Need a Line in the Sand

Here is the tension: pragmatism without consistency is chaos.

On a team, "use your judgment" means five developers make five different judgment calls. One puts validation in the controller, another uses a Form Request, a third validates inside the Action. One developer writes business logic in the controller because "it is only ten lines." The next developer adds fifteen more lines to the same method because "there is already logic here." Six months later, you have a four-hundred-line controller that everyone is afraid to touch — not because anyone made a bad decision, but because nobody made the *same* decision.

The patterns in this book earn their keep in team environments precisely because they are *conventions*. When your team agrees that validation lives in Form Requests, nobody debates where to put it. When business logic always goes in Actions, a new developer knows where to look without asking. The value is not just in the pattern itself — it is in the predictability. Everyone writes code the same way, code reviews focus on logic instead of structure, and the codebase reads like it was written by one person.

This does not mean you apply every pattern from day one. It means you decide as a team which patterns you adopt, and then you follow them consistently. If your team decides that simple validation stays inline and only complex validation gets a Form Request, that is fine — as long as everyone agrees on what "simple" means and the rule is written down somewhere.

The patterns in this book are a starting point for that conversation. Pick the ones that fit your team and your project, agree on when to use them, and then treat that agreement as non-negotiable. Consistency across a codebase is worth more than any individual pattern.

## A Note on AI

You can ask Claude, GPT, or any large language model to write a Laravel controller, and it will. It will even write it well — syntactically correct, following common conventions, probably with a Form Request and an Eloquent query. AI is remarkably good at generating code that works.

But working code is not the same as clean code.

AI does not know your application. It does not know that your team puts business logic in Actions, not controllers. It does not know that your project uses DTOs for data transfer between layers. It does not know that you have a naming convention for Form Requests, or that your models use specific scope patterns, or that your jobs follow a particular structure. It generates code based on statistical patterns from millions of codebases — most of which are not clean.

Here is what I have learned from using AI extensively in my own work: **AI is a multiplier, not a replacement.** If you understand clean code principles, AI accelerates you. You can describe what you want, review what it generates, and correct it quickly because you know what good looks like. If you do not understand these principles, AI gives you more code faster — and more mess faster.

Even the most capable models — Claude Opus 4.6, GPT-5.4, whatever comes next — cannot make architectural decisions for you. They cannot decide whether your application needs an Action or whether the logic belongs in the controller. They cannot judge whether a DTO adds clarity or ceremony to your specific use case. They cannot weigh the trade-offs between a package and a hand-rolled solution in the context of your team and your project. These are judgment calls that require understanding, not generation.

This book teaches you the understanding. Once you have it, use AI as much as you want. Direct it, review it, correct it. You will write better code faster than either you or the AI could alone. But the understanding comes first. Without it, you are just generating confident-looking code that you cannot evaluate.

## What to expect

We start with how to think about clean code — simplicity, naming, dependency injection. Then we move into writing clean Laravel code — controllers, actions, DTOs, jobs, APIs. After that, we cover models and database patterns. Finally, we tie it all together with testing.

Every example uses Laravel 12 and PHP 8.5. No outdated patterns, no deprecated features.
