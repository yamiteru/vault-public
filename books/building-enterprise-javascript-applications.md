# The importance of good code
One thing that separates a good company from a great company is their processes.

## Technical debt
The cummulative effect of making compromises that have negative effect on the codebase in the long-term.

"I'm doing 90% maintenance and 10% development, is this normal"? -> No.

"Most software today is very much like an Egyptian pyramid with millions of bricks piled on top of each other, with no structural integrity, but just done by brute force and thousands of slaves" - Alan Kay, creator of Smalltalk

[[First Mover Advantage]]

Developer can repay the technical debt by refactoring code.

The problem arises when the debt is not paid sufficiently quickly. At some point, the amount of maintenance done on the project is so great that no more features can be added, and the business may opt for a comlete rewrite instead.

In fact, the debt metaphor isn't completely accurate. This is because, unlike a formalized loan, when you incur technical debt, you don't actually know the interest rate or repayment period beforehand.

### Causes of technical debt
1. Lack of talent
		=> Mentoring, code reviews and general training
2. Lack of time
		=> Reducing the scope to something more achievable
3. Lack of morale
	=> Better working environments

Often, the consequences of technical debt are close to immediate, therefore, by rushing, it may actually slow you down within the same development cycle.

In that sense, iincurring technical debt resembles more of a gamble than a loan.

### Consequences of technical debt
- Development speed will slow down
- More manpower (and thus money) and time will nedd to be spent to implement the same set of features
- More bugs, which consequently means poorer user experience, and more personnel require for customer service

### Technical debt leads to low morale
In some cases, those working brownfield projects may even show animosity toward their colleagues who work on greenfield projects.

I can't imagine a developer being happy knowing their skills are becoming less and less relevant.

Typically, developers demand immediate repayment while the (inexperienced) managers would try to push it further down the line.


- Lower productivity
- Lower code quality
- High turnover

### Replaying technical debt throught refactoring
Making our code cleaner without changing the existing behavior.

- Well strucutres
- Well documented
- Succint
- Well formatted and readable

Refactoring is simply a process that moves the current codebase from having a lot of [[Code Smells]] to one that is cleaner.

Developers should be given time to refactor.

Refactoring should be the core part of development process and be included in the time estimates.

### Informing the decision makers
Most decision makers, especially those without a technical background, geratly underestimate the effects of technical debt.

It is important for a professional developer to understand the situation from the desicion maker's perspective.

We can use [[Triple constraint]] to reason about project specs.

Most businesses are limited largely by their time and cost.

**This means that the goal should be a PoC, that can start generating money, and add more features later.**

### Refuse to develop
Refuse further development until refactoring is done (although it's not what a professional developer should do).

Let's suppose you're a doctor and a patient asks you to perform open heart surgery on them in order to relieve a sore throat, what would you do? Of course you'd refuse! Patients do not know what is best for them. that's why they must rely on your professional opinion.

They pay you not to simply code; they pay you because they want you to bring value to their business.

Are your actions beneficial or detrimental to their business?

Business owners also need to trust the advice of their developers. If they do not respect their professional opinion, they shouldn't hire them in the first place.

### Don't be a hero
The developer that commits to those demands is equally at fault.

It is the business owner's, or you manager's, role to get as much out of yiou as possible, But more importantly, it is your duty to inform them of what is and isn't possible.

- You may not actually complete the feature in time
- You've demonstrated to the manager that you're willing to accept these deadlines, so they may set even tighter deadlines next time
- Rushing through code will likely incur technical debt
- Your fellow developers may resent you, since they may have to work overtime in order to keep up with your pace

**Busines owners would rather hear that it's not possible a month in advance than a promise of everything will be done that was not delivered.**

### Defining processes
Good code starts with good planning, design and management, and is maintained by good processes.

It's important to define the [[Definition of Done]].

At the beginning of each sprint, a meeting is held to review pending tasks and select features to be tackled in this sprint. At the end of each sprint, a retorspective meeting is held to review the progress of the sprint and identify lessons that can be learned and applied to subsequent sprints.

The most prominent technique is [[TDD]].