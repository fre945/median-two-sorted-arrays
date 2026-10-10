# Median of Two Sorted Arrays: The Partition Trick That Turns a Merge Into O(log min(m,n))

LeetCode #4 is the problem that teaches you the difference between a solution that passes and a solution that's correct. Merging two sorted arrays and grabbing the middle takes about fifteen lines. The problem statement asks for O(log (m + n)), and a merge gives you O(m + n). With the actual constraints (0 ≤ m, n ≤ 1000, 1 ≤ m + n ≤ 2000), the merge will run and get accepted. That's also why so many people finish it without understanding it.

Here's the thing worth internalizing: this is not a median problem. Strip away the wrapping and it's a binary search problem wearing a median costume. Once you see the partition, the code is short, and the boundary cases stop feeling random.

## What the problem actually asks for

You get two already-sorted arrays, `nums1` of size m and `nums2` of size n, and you return the median of the combined set of numbers. No merging required by the problem — the "combined sorted array" only exists conceptually.

Odd total length → the single middle element. Even total length → the average of the two middle elements.


nums1 = [1, 3],  nums2 = [2]     → 2.0    (merged: [1,2,3])
nums1 = [1, 2],  nums2 = [3, 4]  → 2.5    (merged: [1,2,3,4])


Constraints are small and include zero-length arrays: `0 <= m <= 1000`, `0 <= n <= 1000`, `1 <= m + n <= 2000`, and values from −10⁶ to 10⁶. The arrays are never both empty. LeetCode tags it Array, Binary Search, and Divide and Conquer.

The zero-length case is a real edge case in practice, not a formality. If `nums1` is empty, the answer is just the median of `nums2`. Any implementation that assumes both arrays have at least one element will break on a test that the problem explicitly permits.

## The partition idea, spelled out

Instead of merging, imagine drawing one cut in each array. The cut in `nums1` sits at index `i`, so `nums1[0..i-1]` is on the left side. The cut in `nums2` sits at index `j`, so `nums2[0..j-1]` is on the left side.

Two conditions define a *valid* cut:

1. **The left side holds exactly half the elements.** If the left side contains `(m + n + 1) // 2` elements total, then when the total is odd, the left side has the extra element — which is exactly where the median lives. Writing that as an equation: `i + j = (m + n + 1) // 2`.
2. **Nothing on the left is bigger than anything on the right.** Concretely, `maxLeft1 <= minRight2` and `maxLeft2 <= minRight1`.

When both hold:

- Odd total → median is `max(maxLeft1, maxLeft2)`.
- Even total → median is `(max(maxLeft1, maxLeft2) + min(minRight1, minRight2)) / 2`.

The second equation is the load-bearing one. Because `i + j` is fixed by condition 1, choosing `i` immediately determines `j`. You're searching over a single variable. And the array is sorted, so as `i` increases, `maxLeft1` goes up and `minRight1` goes down — monotonic behavior, which is the exact precondition for binary search.

That leaves `i ∈ [0, m]`, a range of m + 1 candidate cuts, searched in log time. To make that log as small as possible, always binary search the **shorter** array. Swap the inputs at the top if needed. This isn't just an optimization: it's also what keeps `j` valid. With `half = (m + n + 1) // 2` and nums1 guaranteed shorter, `j = half - i` can't fall outside `[0, n]`.

## Walking one example by hand

Take `nums1 = [1, 3, 8]` and `nums2 = [7, 11, 18]`. Total is 6, so `half = 3`, and the median is the average of the 3rd and 4th smallest values.

**Try `i = 0`** → `j = 3`. Left side is `[] + [7, 11, 18]`, so `maxLeft2 = 18`. Right side starts with `nums1[0] = 1`. Check `maxLeft2 <= minRight1`: 18 ≤ 1 is false. `i` is too small.

**Try `i = 2`** → `j = 1`. Left side is `[1, 3] + [7]`, so `maxLeft1 = 3`, `maxLeft2 = 7`. Right side is `[8] + [11, 18]`, so `minRight1 = 8`, `minRight2 = 11`.

Check both conditions: `3 <= 11` ✓ and `7 <= 8` ✓. Valid.

Median = `(max(3, 7) + min(8, 11)) / 2 = (7 + 8) / 2 = 7.5`. Compare against the merged array `[1, 3, 7, 8, 11, 18]` — the two middle values are 7 and 8. Correct.

The direction rule, in one line each:

- If `maxLeft1 > minRight2`, you've taken too many elements from `nums1`. Move the cut left: `hi = i - 1`.
- If `maxLeft2 > minRight1`, you've taken too few. Move right: `lo = i + 1`.
- Otherwise, you've found it.

## Python implementation

python
def findMedianSortedArrays(nums1, nums2):
    # Binary search the shorter array: log(min(m, n)) instead of log(max(m, n))
    if len(nums1) > len(nums2):
        nums1, nums2 = nums2, nums1

    m, n = len(nums1), len(nums2)
    half = (m + n + 1) // 2   # left side size; carries the extra element when total is odd
    lo, hi = 0, m

    while lo <= hi:
        i = (lo + hi) // 2        # how many elements come from nums1 for the left side
        j = half - i              # the rest must come from nums2

        # Sentinels replace the "cut at index 0" and "cut at the end" special cases
        left1  = nums1[i - 1] if i > 0 else float('-inf')
        right1 = nums1[i]     if i < m else float('inf')
        left2  = nums2[j - 1] if j > 0 else float('-inf')
        right2 = nums2[j]     if j < n else float('inf')

        if left1 > right2:        # cut too far right in nums1
            hi = i - 1
        elif left2 > right1:      # cut too far left in nums1
            lo = i + 1
        else:
            if (m + n) % 2 == 1:
                return max(left1, left2)
            return (max(left1, left2) + min(right1, right2)) / 2


Two details in there matter more than they look. First, the infinities. Writing `left1 = nums1[i-1] when i > 0` is the same thing without the sentinel trick, but the sentinel version keeps the comparison logic uniform and removes four near-duplicate branches. Second, `lo, hi = 0, m` — the upper bound is `m`, not `m - 1`, because a cut at index `m` (the entire array on the left side) is a legal partition.

Time: O(log min(m, n)). Space: O(1).

## The bugs that actually cost submissions

Every one of these is a real failure mode, and most of them produce a wrong answer on a case you didn't think to test:

- **Integer division in the even case.** In Python 3, `/` gives a float, so it works. In Java or C++, `(a + b) / 2` on two ints truncates and returns a wrong median. Cast before dividing, or divide by `2.0`.
- **Returning an int instead of a double.** The problem wants `2.5`, not `2`.
- **`hi = m - 1`.** Then the partition that puts all of `nums1` on the left is unreachable, and inputs like `nums1 = [1, 2]`, `nums2 = [3, 4]` can fail depending on the search path.
- **Not swapping to the shorter array.** If you binary search the longer array, `j` can go negative or exceed `n`, and you get an index error or a bad comparison instead of a clean wrong-answer.
- **Forgetting that one array can be empty.** With `nums1 = []`, the loop never runs a meaningful iteration unless the sentinels handle it. The `float('-inf')`/`float('inf')` guards cover this — but only if `j` computed from `half` lands in range, which is why the swap matters.
- **Comparison direction reversed.** `left1 > right2 → hi = i - 1` is easy to flip under interview pressure. Sanity-check the direction with a two-element example before you start reasoning about anything longer.
- **Duplicates.** Equal values are fine here — the conditions use `<=`, so `maxLeft == minRight` is a valid cut. Problems arise only if you write strict `<` on both sides.

If you can explain why each of those is a bug, you understand the algorithm. If you're memorizing the branch order, an interviewer's follow-up question will find the gap.

## The other accepted route: find the k-th smallest

There's a second O(log) family of solutions that skips partitions entirely by reframing the problem: the median is a k-th smallest element query, where `k = (m + n + 1) // 2` for the lower middle and `(m + n + 2) // 2` for the upper middle. For odd totals those two values are identical, so averaging them is always safe — one code path for both parities instead of an if/else.

To find the k-th smallest, you compare the element at position `k/2` in each array. The one that's smaller and everything before it can be discarded, because at least `k/2` elements are smaller than or equal to it. Recurse with a reduced `k`.

AlgoMonster's in-depth write-up for this exact problem takes that route: `findKthSmallest` called twice with those two values of `k`, then averaged, with sentinel `Integer.MAX_VALUE` values when an index would run past the end of an array. It's the same complexity class as the partition version and arguably easier to reason about recursively, though the partition approach is easier to write iteratively without off-by-one headaches.

For reference, the approaches stack up like this:

| Approach | Time | Extra space | Notes |
| --- | --- | --- | --- |
| Merge into a new array | O(m + n) | O(m + n) | Simplest to write, violates the stated requirement |
| Two-pointer walk to the middle | O(m + n) | O(1) | Track the last two values, stop halfway |
| Binary search on the partition | O(log min(m, n)) | O(1) | Iterative, sentinels for the boundaries |
| Recursive k-th smallest | O(log(m + n)) | O(log(m + n)) recursion | Two calls, averaged |

All four are worth writing once. The first two train the problem definition; the last two train the version an interviewer is actually asking for.

## What interviewers push on after you solve it

Getting a clean O(log) solution is roughly half of the signal. The rest comes from what happens next, and the follow-ups are predictable:

- **"What if the data doesn't fit in memory and arrives as a stream?"** The partition approach needs random access to both arrays. A stream breaks that assumption, and you'd reason about a different structure entirely.
- **"Generalize to the k-th smallest element in two sorted arrays."** This is the k-th smallest route above, and it's the most common escalation.
- **"What about three sorted arrays, or a sorted matrix?"** Same partition intuition, more cuts to place.
- **"Why binary search the smaller array?"** Be ready to answer with the bound on `j = half - i`, not just "it's faster."
- **"Walk me through the case where the answer comes entirely from one array."** Inputs like `[1, 2]` and `[100, 101]` are where sloppy boundary handling shows up.
- **"What's the median of a sliding window?"** A different problem (two heaps), often asked right after this one to check you're not pattern-matching on the word "median."

That last follow-up is the reason this problem is worth more than a memorized template. Interviewers use it because the median keyword appears in several unrelated patterns, and the ability to pick the right one under time pressure is exactly what's being tested. The partition trick is a *binary search on a monotonic predicate* — recognizing the family matters more than remembering these twenty lines.

If you want structured practice on that recognition step, AlgoMonster's binary search pattern section breaks the approach into the partition framework, the k-th smallest variant, and the "minimum/maximum that satisfies a condition" style that shows up in a lot of hard problems. 👉 [Read the binary search pattern breakdown on AlgoMonster](https://algo.monster?fpr=li69)

## Where AlgoMonster fits in

AlgoMonster is a text-and-diagram interview prep platform built by ex-Google engineers and competitive programmers. Its approach is pattern-first: instead of grinding a large problem bank, you work through explicit patterns (binary search, two pointers, sliding window, BFS/DFS, DP and so on), each with a conceptual explanation and a reusable code template. The stated curriculum is 500+ lessons across 6 courses, 1,000+ illustrations and 3,000+ editorials, with real questions from major tech companies updated monthly. There's also a decision flowchart system for figuring out which pattern a new problem needs — which is the skill gap that the follow-up questions above are probing.

The subscriptions also throw in a couple of bonus courses at no extra cost, including one on building an LLM from scratch (tokenization, attention, training) and a CS fundamentals course. One caveat from the official page: some specialized courses developed with external partners, such as Quant Interview Prep, are sold separately.

Third-party reviews are pretty consistent about what you're buying and what you aren't. Final Round AI's write-up describes the platform as entirely text-based with no video content and roughly 230 problems versus LeetCode's thousands, and notes the coverage is thin on system design and behavioral prep for senior loops. Lodely's review makes the same point from the other direction: the patterns and templates are the product, the problem count isn't. If you learn by watching someone code on video, this format will feel dry. If you'd rather read a tight explanation and then write code yourself, it's a good fit.

## Pricing

AlgoMonster doesn't do monthly billing. There's a free tier, an annual plan, and a lifetime plan. The Pro Yearly plan currently shows as 50% off its list price, at $99/year billed annually ($8.25/month), and Pro Lifetime at $189 one-time, down from a listed $378.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems; enough to test the teaching style | $0 | — | [Start on the free tier](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant and Speedrun; 500+ lessons across 6 courses; company question sets updated monthly; bonus CS and LLM courses; member discount on live bootcamps | $99/year ($8.25/mo) | Billed annually, cancel or upgrade anytime | [Get Pro Yearly access](https://algo.monster?fpr=li69) |
| Pro – Lifetime | Everything in the yearly plan plus lifetime updates, paid once | $189 | One-time | [Get Pro Lifetime access](https://algo.monster?fpr=li69) |

The annual plan is the sensible pick if you have interviews in the next couple of months and will stop using it after. Lifetime starts paying for itself in year two, which matters if you expect to interview again during a job change two or three years out. Both are well under the cost of a couple of months of coaching, and both are cheaper than a year of LeetCode Premium.

## The short version

The median of two sorted arrays is a binary search problem once you notice that the total size of the left half is fixed, so one cut determines the other. Search the shorter array, check `maxLeft1 <= minRight2` and `maxLeft2 <= minRight1`, handle the boundaries with infinities, and don't integer-divide the even case. That's the whole algorithm.

The part that takes longer than the algorithm is the edge cases, and the part that takes longer than the edge cases is recognizing the pattern quickly enough in a 45-minute interview to spend your time on the follow-ups instead of on remembering the branch order.
