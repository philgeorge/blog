Title: Thoughts on Pair Programming
Published: 26 January 2011
Tags:
- Pair Programming
- Agile
- Teams
---
After many years of talking but frankly very little actual doing, I'm finally putting my money where my mouth is and seriously engaging in pair programming.

![A pair of ducks, not programming](/posts/img/pair-of-ducks.jpg){width="100%"}

Since the whole agile buzz came to my attention about 8 years ago it's always seemed like a good idea. From time to time I've found reasons to "pair up":

- to have help when developing a more complex bit of code
- to acheive "skills transfer" about a code base when one developer is leaving a project
- to help a junior developer get up to speed

But until now, these activities have always been in the minority compared to time sat on my own at the keyboard.

At my current employer, we developers are lucky enough to be trusted to decide for ourselves what tools and technologies we use and together with the project manager we can use whatever methodology suits us. For the new phase of our current project we all agreed to develop in 1-2 week iterations and to do all coding in pairs.

Now I would estimate about 80% of my programming time is with a pair. Previously it would have been about 20%.

There are 6 developers in the team so we can easily split off into 3 pairs, and we make sure to keep rotating who pairs with who. Within the pair we regularly switch who is doing the typing (e.g. person 1 type out an interface, then person 2 write the unit tests for that interface.

We have also tried to regularly swap what the pairs work on. For example, pair 1 will write a repository interface and tests while pair 2 write an service interface and tests. Then we swap over and pair 1 implement the repository so that the other pair's tests pass and vice-versa for pair 2 implementing the service class.

Overall I think this is perhaps the best team programming experience I've had in 16 years as a professional software developer. It fosters a team spirit and cohesion like nothing else.

Things I'm loving about it:

- real-time review of every line of code written
- the non-typer will often call out edge case scenarios or potential bugs during the initial code, that therefore never get left until later or missed until testing
- you learn loads of stuff (programming styles, patterns, APIs, interpretation of requirements, etc) from the person sat next to you
- you improve your ability to explain and justify the code you are writing, and often change it for the better as a consequence
- everyone in the team has worked on most areas of the codebase, or are at least confident about understanding them: there's no more "dave has to fix that bug because he did that bit of code"
- we don't get junior developers kept away from all the tough bits of code, or reduced to only working on boring stuff
- we find ourselves discussing early on across the whole team about the design and structure of the code (e.g. typical responsibilities of a service class, namespaces and assemblies, caching strategies, etc); this gives me great confidence that the overall project will be coded in a much more consistent manner than previous ones

As for any concerns about reduced productivity due to only have half the number of actual hands concurrently typing at any one time, the results so far put that to rest. We have comfortably completed our first two iterations with more functionality than expected.

I know there will be challenges ahead, because this project has very high expectations for the number of features we develop and limited time available, but definitely believe we are now working in the best way to meet these expectations. I hope that this project is the template for all others going forward.

I've usually been lucky enough to work with plenty of smart people. But now I feel like I am finally working in one of those super-motivated, high-functioning, highly-productive teams I've been reading about on blogs over the years.
