# top k frequent elements: the min-heap solution in Python and Java, the bucket sort follow-up, and the pattern behind both

Top k frequent elements looks like a sorting problem. It isn't. It's a counting problem with a selection step bolted on, and almost everything that separates a passing answer from a strong one lives in that second step.

LeetCode 347 is rated Medium. The task: given an integer array `nums` and an integer `k`, return the `k` most frequent elements in any order. The array can run up to 10^5 elements, `k` is always between 1 and the number of unique values, and the answer is guaranteed to be unique.

That last guarantee matters. It means you never have to worry about which of two tied elements to return, only about how cheaply you can find the winners.

## Count first. That part is basically free.

Before anything gets sorted or heapified, you need frequencies. One pass with a hash map gets you there in O(n), and every language has an ergonomic way to do it: `Counter` in Python, `HashMap.merge` in Java, `unordered_map` in C++.

python
from collections import Counter

freq = Counter(nums)   # {1: 3, 2: 2, 3: 1} for [1,1,1,2,2,3]


After this line, the problem changes shape. You no longer care about the original array. You care about `m` key-value pairs, where `m` is the number of unique elements, and you want the `k` pairs with the largest values. The original array's size is irrelevant from here on, which is exactly why the choice of selection method matters so much.

## Four ways to pick the top k, and what each one costs

| Approach | Time | Extra space | What it's actually good for |
| --- | --- | --- | --- |
| Frequency map + full sort | O(n + m log m) | O(m) | Fastest to write; fine when m is small or you need the full ranking anyway |
| Min-heap of size k | O(n log k) | O(m + k) | The default interview answer; wins clearly when k ≪ m |
| Bucket sort by frequency | O(n) | O(n) | When you want linear time and frequencies are bounded by n |
| Quickselect | O(n) average, O(n²) worst | O(m) | Rare in interviews; useful to know why the worst case exists |

`n` is the array length, `m` is the number of unique elements. Note that `m ≤ n`, and for a typical test case `m` is much smaller, which is why a plain sort often passes comfortably. That's also why "it passed" is not the same as "it's the right answer to give."

## The heap version, step by step

The trick is a **min**-heap, not a max-heap. That inversion trips people up, so it's worth walking through.

Take `nums = [1,1,1,2,2,3]` and `k = 2`. Frequencies: `1 → 3`, `2 → 2`, `3 → 1`.

1. Push `(3, 1)` (frequency, value). Heap: `[(3,1)]`. Size 1 ≤ k, keep it.
2. Push `(2, 2)`. Heap: `[(2,2), (3,1)]`, ordered so the smallest frequency sits at the root. Size 2 ≤ k, keep both.
3. Push `(1, 3)`. Heap temporarily holds three items, then `3 > k` forces a pop of the minimum frequency, which is `(1, 3)`.
4. The heap now contains `(2,2)` and `(3,1)` — the two highest frequencies. Extract the values: `[1, 2]`.

The reason a min-heap works is that its root is always the weakest survivor. Every time the heap overflows, the weakest element gets evicted, so the strong ones can never be pushed out by a low-frequency newcomer. A max-heap of size k wouldn't give you a cheap way to answer "who's the weakest one here?"

python
import heapq
from collections import Counter

def topKFrequent(nums, k):
    freq = Counter(nums)
    heap = []
    for num, count in freq.items():
        heapq.heappush(heap, (count, num))
        if len(heap) > k:
            heapq.heappop(heap)        # evict the least frequent
    return [num for count, num in heap]


Java needs an explicit comparator, since `PriorityQueue` is a min-heap by default and you want the smallest *frequency* at the top:

java
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);

    PriorityQueue<int[]> heap = new PriorityQueue<>((a, b) -> a[0] - b[0]); // by frequency
    for (Map.Entry<Integer, Integer> e : freq.entrySet()) {
        heap.offer(new int[]{e.getValue(), e.getKey()});
        if (heap.size() > k) heap.poll();
    }

    int[] res = new int[k];
    for (int i = 0; i < k; i++) res[i] = heap.poll()[1];
    return res;
}


Python has a shortcut that hides all of this: `Counter(nums).most_common(k)` internally does the same shape of work and returns `(element, count)` tuples in descending frequency order. It's a legitimate answer, and worth mentioning in an interview only if you can also explain what it's doing underneath.

## The bucket sort version, when you want linear time

Frequencies are bounded. No element can appear more times than the array is long, so the frequency values live in the range 1 to n. That makes them perfect bucket indices.

python
def topKFrequent(nums, k):
    freq = Counter(nums)
    buckets = [[] for _ in range(len(nums) + 1)]   # index = frequency
    for num, count in freq.items():
        buckets[count].append(num)

    res = []
    for count in range(len(buckets) - 1, 0, -1):   # high frequency first
        for num in buckets[count]:
            res.append(num)
            if len(res) == k:
                return res


Two things break this code if you're careless. The buckets array has to be `len(nums) + 1` long, not `len(nums)`, because the top index is the maximum possible frequency. And the reverse loop starts at the last index and skips empty buckets — if you assume every frequency is represented, you'll crash on inputs where they aren't.

Time complexity is O(n) with O(n) space. It's the answer that signals you noticed the frequency bound is a property of the input, not something you have to sort around.

Quickselect is the fourth option: partition the frequency list around a pivot the way `nth_element` or `quickselect` does, recurse only into the side containing rank k, and stop. Average O(n), worst-case O(n²) on adversarial pivots. It shows up in interview discussions more often than in production code.

## Edge cases that actually get tested

- **k equals the number of unique elements.** Every unique value is in the answer. The heap version works untouched; the bucket version needs the loop to survive returning everything.
- **All elements distinct.** Every frequency is 1. Any k of them is a valid answer, and this is where a bad tie-handling assumption blows up.
- **Ordering isn't part of the contract.** For 347, `[1, 2]` and `[2, 1]` are both accepted. Say that out loud in an interview; it saves you from writing unnecessary comparison logic.
- **Space claims.** "O(k) extra space" is a common slip. The heap is O(k), but the frequency map is O(m) and doesn't disappear. Total is O(m + k).
- **Integer overflow.** Not a Python problem, but in Java and C++ be careful when you later sum frequencies or distances in the related problems.

## The follow-ups that decide the interview

347 is a doorway. The variations test whether you learned the pattern or memorized an answer.

**Top K Frequent Words (LeetCode 692)** adds an ordering rule: same frequency means alphabetical order first. Now the sort key has to be a tuple like `(-count, word)` in Python, or a comparator that falls back to `compareTo` in Java. The moment ties stop being free, a plain heap needs care.

**K Closest Points to Origin (973)** is the same skeleton with a different metric: heap on squared distance, and skip the square root entirely because comparing squares preserves order.

**Kth Largest Element (215)** and **Kth Largest in a Stream (703)** are the single-target and streaming versions. The streaming one forces the min-heap-of-size-k idea into full view, because you can't look at the data twice.

**Amazon's OA variant** — Top K Frequently Mentioned Keywords — pushes further. Each keyword counts at most once per review, which means you build a set of matches per review before incrementing, and words are matched case-insensitively on word boundaries. Then ties break alphabetically, so the heap holds inverted comparisons. The AlgoMonster editorial for this variant walks through that regex-plus-heap combination in Python and Java.

The signal to remember: **if the problem mentions k, a top-k, or the kth anything, a heap is probably in play.** The follow-up question is always the same one — do you need the full ranking (sort), only the k best (heap), or can the keys be counted directly (bucket)?

## Practicing this so it transfers to the next problem

Solving 347 once teaches you 347. The transferable skill is recognizing "k plus frequency" and immediately knowing which of the four approaches the constraints allow, then explaining the choice under pressure.

That's the gap a pattern-organized resource is supposed to close. AlgoMonster builds its curriculum around explicit patterns, and the priority queue/heap pattern is one of them — reviews of the platform consistently describe 48 patterns covering roughly 231 lessons and problems, with the official material advertised as 500+ lessons across 6 courses, 1,000+ illustrations and 3,000+ editorials, all written by engineers from Google and competitive programming backgrounds.

The editorial for this exact problem is public, and its structure is the useful part: frequency map, then heap, then extraction, with the complexity tradeoff stated rather than implied. If 347 itself is what you're stuck on, 👉 [start with the top k frequent elements walkthrough on AlgoMonster](https://algo.monster/liteproblems/347?fpr=li69) and compare its reasoning against yours.

Two features matter more than the problem bank. Decision flowcharts walk you from "the input is unsorted and I keep asking about k" to the likely pattern, which is the step people fumble in live interviews. Speedrun puts a timer on mixed problem sets, which is closer to real interview conditions than untimed practice.

What it doesn't have is worth knowing too: there are no video walkthroughs (everything is text and diagrams), the problem count is far smaller than LeetCode's, and system design coverage is described by reviewers as introductory rather than senior-level.

## What AlgoMonster costs

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems | $0 | — | [Start with the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro Yearly | All patterns, lessons and problems, unlimited AI assistant, unlimited Speedrun, company questions updated monthly, member discount on bootcamps | $99/year (list $198; ~$8.25/mo billed annually) | Billed annually, cancel or upgrade anytime | [Get Pro yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro Lifetime | Everything in the yearly plan plus lifetime updates, one payment | $189 (list $378) | Billed once | [Get Pro lifetime](https://algo.monster/subscribe?fpr=li69) |

Two plan notes. There is no monthly option, so it's annual or lifetime, nothing in between. And there's no team or enterprise tier — this is built for one person studying on their own. Sale pricing moves around, so check the page rather than trusting a number from a review written months ago, including this one. You can 👉 [see the current plans and any active discount here](https://algo.monster/subscribe?fpr=li69).

## Is it worth paying for, given you could just read a free editorial?

For this one problem, no. There are free editorials for 347 all over the internet, including AlgoMonster's own, and if `top k frequent elements` is your only blocker, you can be done in twenty minutes without spending anything.

The paid plan makes sense if you want the surrounding structure: the heap pattern taught next to the problems that reuse it, flowcharts to shorten your pattern-recognition step, and timed sets to practice execution under pressure. The lifetime option breaks even against roughly two years of annual renewals, so the honest decision is whether you expect to interview more than once in the next few years. If you're preparing for one job search that ends in six weeks, the yearly plan is the smaller commitment.

## Quick answers

**Is O(n log k) good enough?** Yes, and it's usually the expected answer. Bucket sort's O(n) is the upgrade you mention when asked whether linear time is possible.

**Does the answer order matter?** Not for 347. It does for 692 and for Amazon's keyword variant, where ties resolve alphabetically.

**Does the free tier cover this problem?** It gives you a sample of the content. The editorial page for 347 is readable, but the full pattern library and Speedrun sit behind the paid plans.

**Can I cancel?** The plan page states cancel or upgrade anytime on the yearly option.

**Is the heap really the pattern, or just this problem?** It's the pattern. Kth largest, k closest points, streaming top-k, top k words, top k keywords: same structure, different comparison key and different tie-breaking rule. Learn the structure once and the variations become small edits instead of new problems.
