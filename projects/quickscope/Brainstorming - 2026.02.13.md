I want to create a system which tracks the "life" of all code even across languages.

So for example if a config for something has been saved in JSON and then refactored into TypeScript as a hardcoded variable and then moved from constants somewhere else and made dynamic I should be able to see this history of this particular piece of code when I click/hover on it.

The system should work on as granular level as possible.

The smallest level has to still be meaningful.

The level at which code is meaningful will definitely have to be smaller than a function.

But where exactly this threshold lies is hard to pinpoint.

For example we could argue that even `a + b` in the given context is its own entity which can then morph into `a + b + c` and then into `(a + b) + (c - d)`.

If I click/hover on the final code I should be able to see that it originated simply as `a + b` and then it was refactored in a commit `feat: make stats more robust` into `a + b + c` and then later in another commit it was refactored in the current state.

I think figuring out the smallest possible resolution is somewhat tied to the "resurrection" feature.

Imagine there's a simple code like `emailService.send({ subject: "Happy Birthday!", body: "...", to: "..." })` which is triggered inside some larger code A.

This code gets deleted because someone says they don't like getting these emails.

6 month pass and a new exec comes up with an idea of sending happy birthday emails to customers to show that we care about them (which was the initial idea before it was deleted in the first place).

Maybe variable/function names and APIs have changed in those 6 months so the new code looks like this `email.to("..").subject("Happy Birthday {name}!").body("..").send()` and is not inside some larger code B (not A - for whatever reasons).

Now for this to work I don't see any other way to do this than creating embeddings from each piece of code (still don't know what "piece of code" means) and doing topK search for all newly created pieces of code against all deleted pieces of code.

But I fear this would not be enough since the general meaning can be somewhat similar but it might not be *the same thing* that was in the code before.

Here *the same thing* means same in the semantic meaning so in the "meaning" meaning.. true meaning.

I cannot imagine a static and non-AI system where this could be resolved with high accuracy so I guess the only was to do this would be to use some kind of AI.

This obviously raises the cost quite a bit but probably the future of code history *is* smart anyways so it would become a thing either way.

The reason why static methods don't work here is that I can drastically change the "shape" (AST) of the code AND also change variable names so both simple textual similarity and AST change similarity would be very low.

So I guess the PoC of something like this would completely ditch static methods altogether and instead heavily rely on LLM embeddings and reasoning to figure out what kind of code change the current diff is and how it ties to the enormous pool of temporal (code) nodes in what I imagine would be a hybrid graph and vector database.

The very reason why I even want to make this is to create a better code intelligence platform which will use code metrics, relationships and history to accurately pinpoint code hotspots worth fixing.

There are platforms that already allow this (namely CodeScene) but they're very dumb and crude in the way they figure these things out.

They simply compute things like cyclomatic complexity for functions over the timeline of the repository which is a good start but it cannot answer many questions because fundamentally it cannot provide the necessary context.

Getting this context requires the creation of a complex and very detailed graph of temporal relationships.

We need to know when each piece of code was born, out of what it was born (e.g. "thin air" - new feature or merged from 2 different pieces of code or split from 1 piece of code into 2 where the current piece of code is one of those 2).

Another problem with Git commit history is that commit messages completely rely on humans which means that the reliability of a system based on commits is proportional to the quality of those commits.

This is tragic in real life because most people and teams don't produce granular enough commits with granular enough descriptions.

This means that my tool should completely ignore git messages and derive the meaning/intend purely from the meaning of individual pieces of code in their specific and current context in respect to the code changes created.

(maybe commit messages might be used only to give a bit more info about the "why" - but most commit messages don't have this info since it generally lives in project management tools like Jira)

This leads me to another thought which is that Git history is very tightly coupled with the PM tool history.

I imagine that my tool would gather info for each set of commits (PR) from the PM tool in order to make more informed decisions about the intend behind the code changes.

In a philosophical sense code changes are mere manifestations of requirements which are typically stored in PM tools and chat apps.

What this means is that the tool should try to use all available data sources it has at its disposal at any given time to gather all information about the "why" behind every code change.

(In a completely ideal world all of this info would be stored in this app and not spread across different apps but unfortunately that's not the case and probably won't be so I have to be pessimistic here)

I imagine that since this should be an intelligent system it should simply ask or at least explicitly list assumptions when it's not sure about the intend behind code changes.

Recently I've been watching a lot of game programming videos about low-level optimization and I've found a parallel with this project in at least one way.

I can't remember what the term was but it's an optimization strategy for ray/path tracing which divides the otherwise possibly infinitely detailed space into sections/chunks which means that the ray doesn't have to check 1000s of points in space but instead just a few 10s.

Similar idea is also found in particle simulation (like fluid or sand simulation) where there're hybrid solutions where the "true" simulation is done for the particles on the surface and an "approximate" simulation for the rest.

So when it comes to my "piece of code" chunking logic I think I should do it dynamically.

So for example when a file with several functions is first created theoretically we could say that this whole file is a "piece of code".

But later as we add more things into the code, fix bugs, change signatures, refactor the code, merge other code into this one, split this code into other pieces of code, etc. we start dividing and splitting this huge "piece of code" into smaller pieces.

So for example if I move one of those functions out of the file it would create a "piece of code" for the new function as well as for the new state of the first file which now has one less function.

Both of these new "pieces of code" are pointing at the very first one as their source/parent.

I imagine each of these "arrows" (edged in a graph) has a type and maybe some metadata.

For example the extracted function would have an arrow of type `EXTRACTED` and the remaining set of functions would maybe have a type `SPLIT`.

I imagine a single code change could produce a new node with X edges.

For example if I both extract the function AND refactor it (maybe change its signature and make error handling better) it should point back at the original code with edges of type `EXTRACTED`and `REFACTORED`.

Theoretically each edge could also have "reason" metadata which in the case of `EXTRACTED` would maybe say "index.ts was getting too big" and `REFACTORED` would maybe say "signature was changed to support 'from' field because we need to differentiate between different senders" and "error handling has been improved because in production users were seeing cryptic error messages which led to confused users".

(maybe it could be several `EXTRACTED` edges each with a different `reason` or a single edge with an array of reasons) 