# first bad version: the O(log n) boundary template, the two bugs that break it, and where to drill it

Most people who look up "first bad version" fall into one of two groups. Group one is stuck on the actual LeetCode 278 solution and wants the code. Group two already wrote something that passes and still feels shaky about *why* the boundary moves the way it does — which is the version that bites you in a 45-minute interview, when the interviewer changes one line and asks what happens now.

This post covers both: the problem, the two template variants that solve it, the two bugs that show up again and again, a full walkthrough, and the sibling problems that use the exact same idea.

## What First Bad Version actually asks

You have `n` versions labelled `1` through `n`. Somewhere in that sequence a version went bad, and because every version builds on the previous one, everything after the first bad version is bad too. You get an API `bool isBadVersion(version)` and you implement `firstBadVersion(n)` so it returns the earliest bad version. The scoring pressure is the last line of the prompt: minimize the number of API calls.


n = 5, first bad version = 4

isBadVersion(3) -> false
isBadVersion(5) -> true
isBadVersion(4) -> true
answer: 4


Constraints: `1 <= bad <= n <= 2^31 - 1`. That upper bound is not decoration. It's the number that breaks the naive midpoint formula in Java and C++, which we'll get to.

LeetCode tags it Easy. That tag is a little misleading. The problem is easy if you know the "first true index" template. If you don't, it's where you spend twenty minutes shuffling `left` and `right` until the test cases stop failing.

## The whole problem in one sentence: find the first true

Every version from 1 to `bad - 1` returns `false`. Every version from `bad` to `n` returns `true`. So the API is a monotonic predicate, and the array of answers looks like this:


[false, false, false, true, true, true]
                 ↑
        the first true — this is the answer


Once you see it that way, First Bad Version stops being a special problem. It's a boundary search. Same shape as "first index where the value is at least X", "last index where the condition holds", "smallest capacity that finishes the job in time". You write a feasible function, then binary search for the boundary instead of for a value.

That distinction matters in interviews. Value-based binary search (`arr[mid] == target`) is what most tutorials teach, and people try to force it here — checking whether `isBadVersion(mid)` is true and then looking at `mid - 1` to see if it's the *first* one. It works, but it costs extra API calls and it's fragile at `mid == 1`.

## Template A: while left < right

The compact version most people end up writing:

python
def firstBadVersion(self, n: int) -> int:
    left, right = 1, n
    while left < right:
        mid = left + (right - left) // 2
        if isBadVersion(mid):
            right = mid        # mid could be the answer, keep it in range
        else:
            left = mid + 1     # mid is good, so the answer is strictly after it
    return left


Two things carry all the weight here:

- When `mid` is bad, you set `right = mid`, not `right = mid - 1`. You haven't checked `mid - 1` yet, so you can't rule it out.
- The loop is `left < right`, which means it terminates when both pointers land on the same index. That index is the answer, so you return `left`.

The integer overflow fix also lives here: `mid = left + (right - left) // 2` instead of `(left + right) // 2`. With `n` up to `2^31 - 1`, adding two large values in a 32-bit `int` wraps into a negative number, and the loop spins with a negative `mid`. On LeetCode this shows up as a Time Limit Exceeded verdict on the largest test case rather than a wrong answer, which is exactly why it confuses people — the code looks correct.

Python is immune to that particular bug (integers are arbitrary precision), so a Python solution can get away with `(left + right) // 2`. If you're writing Java, C++, C#, or Go in the interview, don't.

## Template B: while left <= right with a recorded answer

The other idiomatic version uses an inclusive loop and tracks the best answer found so far:

python
def firstBadVersion(self, n: int) -> int:
    left, right = 1, n
    first_true = -1
    while left <= right:
        mid = left + (right - left) // 2
        if isBadVersion(mid):
            first_true = mid
            right = mid - 1    # look for something even earlier
        else:
            left = mid + 1
    return first_true


This one is more verbose but harder to get wrong, because the loop condition and the pointer updates are the same as a standard value search. If you're going to fumble an off-by-one under pressure, you're more likely to fumble Template A.

AlgoMonster pushes this structure hard — its binary search lesson is built around a single reusable "first true index" template with the feasible function plugged in, and its write-up of First Bad Version deliberately notes the three differences from other variants: it returns a recorded index instead of `left`, it uses `<=` not `<`, and it uses `right = mid - 1` when the predicate is true. If you've been collecting templates from five different sources and they keep contradicting each other, that's why.

## The walkthrough: n = 8, first bad version is 5

Versions look like `[F, F, F, F, T, T, T, T]`. Using Template B:

| Step | left | right | mid | isBadVersion(mid) | What happens |
| --- | --- | --- | --- | --- | --- |
| 1 | 1 | 8 | 4 | false | `left = 5`, `first_true` still -1 |
| 2 | 5 | 8 | 6 | true | `first_true = 6`, `right = 5` |
| 3 | 5 | 5 | 5 | true | `first_true = 5`, `right = 4` |
| — | 5 | 4 | — | loop ends (`left > right`) | return 5 |

Three API calls for eight versions. A linear scan from version 1 would have taken five before it hit the bad one, and up to `n` in the worst case.

## How many calls does it actually take?

Binary search halves the range each time, so the call count is about log₂(n). For the maximum `n = 2,147,483,647`, that's roughly 31 calls. A loop from version 1 upward in the worst case is 2,147,483,647 calls — which is why the "minimize API calls" instruction is in the prompt at all. If your solution iterates from 1 and checks each version, it's correct and it will still fail on time.

Complexity: O(log n) time, O(1) space. Say both out loud in the interview; it costs you two seconds and it's part of the expected answer.

## Edge cases worth checking before you submit

- `n = 1`, `bad = 1` — the loop never runs in Template A (left == right from the start), and Template A returns `left = 1`. Template B runs once and records `first_true = 1`. Both fine, but only if your return statement is right.
- `bad = n` — everything is good except the last version. Template A converges to `n`.
- `bad = 1` — everything is bad. Both templates converge to 1. This is where the "check `mid - 1` and `mid` separately" approach gets awkward, because `isBadVersion(0)` doesn't exist as a valid version.
- The 32-bit maximum — see the overflow note above.

## The same idea, wearing different clothes

Once the first-true template is in your head, a pile of "medium" problems collapse into one line of thinking. The feasible function changes; nothing else does.

- **Koko Eating Bananas** — smallest speed `k` that finishes within `h` hours. Feasible: "finishes in time?"
- **Sqrt(x)** — largest integer whose square is at most x. Feasible: "mid² ≤ x?"
- **Search Insert Position** — first index whose value is ≥ target.
- **Find Minimum in Rotated Sorted Array** — boundary on the rotation point.
- **Split Array Largest Sum / capacity-shipping problems** — smallest capacity that satisfies a constraint.

This is the argument for studying patterns instead of one problem at a time, and it's the pitch every structured prep platform makes. The honest version of it: the template is only half the skill. The other half is looking at a fresh problem statement and deciding that the answer is a boundary at all, which takes practice on varied problems — not on First Bad Version twenty times.

## Practising it where it sticks

First Bad Version is a good drill for one specific habit: writing the loop, the midpoint, and the pointer updates without second-guessing yourself. If your failure mode is "I understand the explanation but my code is wrong on the first try", repetition on variants helps.

👉 [read AlgoMonster's full walkthrough of the First Bad Version pattern](https://algo.monster/liteproblems/278?fpr=li69)

AlgoMonster is a text-and-diagram interview prep platform built by ex-Google engineers and competitive programmers. Its structure is a set of coding patterns rather than a giant problem list — the company's own framing is that interviews draw on a small set of repeatable patterns, and the platform pairs each one with a reusable template, a decision flowchart for identifying which pattern a problem needs, and practice problems with editorials. Its library covers roughly 48 patterns, 231 lessons and problems, 670+ illustrations, and solutions in six languages including Python, Java, C++, and Go. Binary search is one of the early patterns, and First Bad Version sits inside it as the canonical "first true index" example.

There's no video content at all. Every explanation is text plus interactive diagrams, and the practice environment includes an in-browser editor and a timed "Speedrun" mode. If you learn better from watching someone code, that design choice is a real drawback rather than a preference thing.

The common criticism in reviews is coverage outside DSA: system design content is introductory, behavioral prep is thin, and there's no live mock interview or mentorship. Reviews also note the problem volume is small next to LeetCode's three thousand plus, so a lot of candidates end up using AlgoMonster for the frameworks and LeetCode for volume.

## What it costs right now

AlgoMonster has no monthly plan — the choice is annual or lifetime, and there's a free tier that both third-party comparisons and AlgoMonster's own page describe as limited access to sample content. Here's the plan structure as shown on the subscribe page:

| Plan | What you get | Price | Billing | Sign up |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons and problems — enough to judge the text-and-diagram format | $0 | No card | [Open the free tier](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company questions updated monthly; 500+ lessons across 6 courses; 1,000+ illustrations; 3,000+ editorials; 12 months of access; member discount on live bootcamps | $99/year ($8.25/mo billed annually) | Annual, cancel or upgrade anytime | [See the Pro – Yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan, plus lifetime updates and an internal referral to a top tech company for qualified candidates | $189 one-time | One payment, no renewal | [See the Pro – Lifetime plan](https://algo.monster/subscribe?fpr=li69) |

A warning about the numbers you'll find elsewhere. AlgoMonster runs sales constantly, and the subscribe page has been captured showing deeper promotional pricing (a yearly rate around $69.99 and a lifetime price around $89) alongside the standard $99 / $189 figures. Meanwhile a few review sites still quote $459 for lifetime access — that came from an earlier pricing structure and no longer matches the page. Treat the checkout total as the only reliable number, and don't trust a blog post that quotes a lifetime price without a date on it.

What you see in the table above is what the page showed at the time of writing; the link applies whatever promotion is live, and partner and creator codes for an extra percentage circulate widely, so the total can land below $99 / $189.

## Yearly or lifetime, if you're buying

The break-even math is simple enough to do in your head. If you're preparing for one interview loop in the next two or three months, the yearly plan does the job and lifetime is money you may never touch again — AlgoMonster's curriculum is designed to be finished in about four to eight weeks. If you expect to job-hunt again in two or three years, the lifetime plan costs less than two years of the annual plan and includes future updates, so it's the cheaper option over the long run.

Neither makes sense if your gap is somewhere else. AlgoMonster assumes you can already write working code in one language; it isn't a programming fundamentals course, and it isn't where you should go to learn what a hash map is. Same if your interview loop hinges on senior-level system design.

👉 [check the current AlgoMonster pricing and start with the binary search pattern](https://algo.monster/subscribe?fpr=li69)

## Quick answers

**Does First Bad Version have to use binary search?** Functionally no — you could scan from version 1 and get the right answer. The problem explicitly asks you to minimize API calls, and a scan is O(n) against roughly 31 calls at the maximum input size, so binary search is the expected solution.

**Why does my Java solution time out instead of failing a test?** Almost always the midpoint: `(left + right) / 2` overflows a 32-bit int when `n` is near `2^31 - 1`, producing a negative `mid` and a loop that never narrows. Use `left + (right - left) / 2`.

**`while left < right` or `while left <= right`?** Both are correct as long as the pointer updates match the loop condition. Mixing them — `while left < right` with `right = mid - 1` — is where the answer gets skipped and you return a version that's one too high.

**Is it worth memorizing the template?** The template saves you from writing five lines of index shuffling under time pressure. What the interviewer is actually testing is whether you recognise a monotonic predicate and reach for a boundary search. Learn it as one pattern with a swappable feasible function, and Koko, Sqrt(x), and Search Insert Position all come along for free.
