We're building AI code review platform/tool/SaaS.

AI code reviewer should *ask* if it's not sure about something.

It should try to get info from all available sources including the Git, the PM tool, special doc tool.

It should auto-apply trivial suggestions.

This connect to a more broad goal which is - do not annoy and spam the human.

Github UI and commenting flow is limited so ideally devs should move to our tool for managing their PRs including chatting with agents and giving them more info.

This tool should keep a memory of the project/s and update it with any new info from any source.

This memory should *not* be destructive meaning we should be able to see what was true at any given time but is not anymore as well and as conveniently as the currently true info.

The tool should have context of the whole repo (or possibly even repos - cross-repo knowledge).

The tool should absolutely prioritize quality over quantity so it's OK if we spend more tokens on each code change if it leads to higher number of caught bugs and lower percentage of false positives/negatives.

The SaaS will *not* be seat priced - instead it will be priced completely based on the usage.

The tool will use advanced static analysis and many language-specific quality tools/linters/parsers/etc. to inform the AI agents so they don't have to spend tokens on identifying these simple issues.

The tool will be just a small part in a big ecosystem/app. The ultimate goal is to build a "AI-only software agency" which means it should be at least as good as a professional human-only software agency filled with senior developers both for creating new software as well as maintaining pre-existing software. This code review part is the first step. Next one will be "minions" which is a CRON-like mechanism which looks for hotspots in the codebase and tries to fix it using AI agents and it does this e.g. once every 12 hours.

The AI agent will actually be several AI agents where each one has a different focus/personality/goal and they have a "debate" (actually critique each others plans in 1-2 turns) and then the "judge" decides what to do.

At any point in time if the AI agent is not sure about something potentially destructive or at the very least annoying (it takes real human time to review and then decline the changes and/or try to correct the AI which is a waste of time and money) it should always ask which is very different from current AI tools that operate on assumptions which are pretty often wrong.

Another child-project of this project will be some kind of semantic search for IDEs where the dev (or their AI agent) can just type "React popup/modal component" and it will correctly find the code that the dev was looking for (not just file but the closest indexed parent which will at the very least be function-level but ideally I would like internally for all of these tools to do semantic scope-level so ifs/loops/blocks/etc.).

Eventually in the next version of this project there should also be "pair programmer" which watches as you (or your AI agent) code, predicts what you're going to do next and tries to catch bugs before you even type them or spend time on writing the code that would produce them (ideally I would like this pair programmer buddy to have different personas where e.g. one could straight away suggest or even apply fixes but a different one could always be kind of a "therapist" that just asks questions and forces the dev to think and solve on their own).

Maybe not in the first version but definitely in later versions all of these apps will utilize the complete Git history, not just the current state, to make better and more informed decisions and provide both the system and the user with more and better metrics/stats.

Eventually there will be a sub-project for grouping uncommited changes into logical groups and creating commit messages for them (again a part of the IDE experience).

Also a nice followup would be a some kind of changelog but with different views - at the very least a technical and non-technical (business value) view which should automate reports.

The code review (or any other) tool should NEVER hallucinate (realistically as little as possible) so it should always triple-check docs, source, etc.

When doing the review it should trace back all variables/functions/classes from the changed code across all levels of the codebase to understand exactly what/where/how/why it does and how this change will change that.

Maybe later versions might use symbolic reasoning on top of static code analyses.

Internally the reviewer should tag each logical change (not necessarily a single hunk) with its type (refactor, new feature, bug fix, doc, etc.) and add a small message which describes the WHY - this will later allow for temporal understanding of the codebase *and* better and more accurate temporal code-change search.

The AI code review agent should have "internal monologue" which works as quality gate to not introduce noise.

There has to be a knowledge graph behind the scenes most likely based on AST or more broadly the structure of the code (maybe we could use GNNs?) - tribal knowledge retention.

The code review tool should not analyze *only* the Git history but also the Github PR history to acquire already present tribal knowledge.

The reviews have to be personalized based on the target dev experience (in general and in regards to the changed code) - maybe also the Socratic method instead of straight up telling devs what they've done wrong.

The code review agent should *not* give opinions - only facts - because AI *cannot* have opinions.

The code reviewer should use a set of models instead of using a single model to introduce diversity.

In the future human reviews will be a liability because humans are unpredictable and slow - this product should bring the future to the present.