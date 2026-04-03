DSA Interview Roadmap for MAANG Preparation
Here's your structured path from zero to interview-ready. Every topic is tagged by interview frequency and includes the kind of problems you'll actually face.

Phase 1: Foundations (The Language of Problem Solving)
Without these, nothing else makes sense. Most candidates rush past this and regret it.
🔴 Must-Know:

Time & Space Complexity (Big O) — How to measure whether your solution is fast and memory-efficient enough. Problems: "Can you optimize this?", "What's the complexity?" — asked about every single solution you write.
Arrays — Contiguous blocks of memory; the most common data structure in interviews by far. Problems: Rotate array, find duplicates, max subarray, merge intervals.
Strings — Character arrays with their own set of manipulation tricks. Problems: Reverse words, valid anagram, longest substring without repeating chars.
Hash Maps / Hash Sets — Key-value lookups in O(1); the single most useful tool for optimizing brute force. Problems: Two sum, group anagrams, frequency counting, subarray sum equals K.
Two Pointers — Using two indices moving through data to avoid nested loops. Problems: Container with most water, three sum, remove duplicates from sorted array.
Sliding Window — A moving window over a sequence to track a running condition efficiently. Problems: Max sum subarray of size K, longest substring with K distinct chars, minimum window substring.

🟡 Good-to-Know:

Prefix Sum — Precomputing cumulative sums so any range-sum query becomes O(1). Problems: Subarray sum equals K, range sum queries.
Bit Manipulation — Using binary operations for clever tricks like finding single numbers or toggling flags. Problems: Single number, counting bits, power of two.


Phase 2: Core Data Structures (Your Interview Toolkit)
These appear in 60–70% of MAANG coding rounds.
Linear Structures:

🔴 Linked Lists — Nodes connected by pointers; tests your ability to manipulate references without losing data. Problems: Reverse a linked list, detect cycle, merge two sorted lists, LRU cache.
🔴 Stacks — Last-in-first-out; great for tracking "most recent" state. Problems: Valid parentheses, daily temperatures, min stack, evaluate reverse polish notation.
🔴 Queues & Deques — First-in-first-out and double-ended variants for ordered processing. Problems: Sliding window maximum, implement queue using stacks, rotting oranges.

Non-Linear Structures:

🔴 Binary Trees — Hierarchical nodes with at most two children; a massive interview category. Problems: Max depth, level order traversal, lowest common ancestor, serialize/deserialize.
🔴 Binary Search Trees (BST) — Binary trees with sorted order property enabling O(log n) search. Problems: Validate BST, kth smallest element, inorder successor.
🔴 Heaps / Priority Queues — Partially sorted structures that give you the min or max in O(1). Problems: Top K frequent elements, merge K sorted lists, find median from data stream.
🔴 Tries (Prefix Trees) — Tree-like structures optimized for string prefix operations. Problems: Implement trie, word search II, autocomplete system.
🔴 Graphs — Nodes and edges representing relationships; the most versatile and feared topic. Problems: Number of islands, clone graph, course schedule, shortest path.

🟡 Good-to-Know:

Monotonic Stack/Queue — A stack that maintains increasing or decreasing order for "next greater/smaller" problems. Problems: Next greater element, largest rectangle in histogram.
Union-Find (Disjoint Set) — Efficiently tracks which elements belong to the same group. Problems: Number of connected components, redundant connection, accounts merge.


Phase 3: Core Algorithms (The Thinking Patterns)
This is where you learn how to approach problems, not just what data structure to use.
Searching & Sorting:

🔴 Binary Search — Halving the search space each step; far more versatile than just "find element in sorted array." Problems: Search in rotated sorted array, find first/last position, koko eating bananas, median of two sorted arrays.
🔴 Sorting Algorithms — Understanding merge sort and quicksort internals (not just calling .sort()). Problems: Sort colors, merge intervals, custom sort comparators.

Recursion & Backtracking:

🔴 Recursion — Breaking a problem into smaller identical subproblems; the foundation for trees, graphs, and DP. Problems: Every tree/graph problem, generating parentheses, power function.
🔴 Backtracking — Recursion + undo; systematically trying all possibilities and pruning bad paths. Problems: Permutations, combinations, N-queens, sudoku solver, word search.

Graph Algorithms:

🔴 BFS (Breadth-First Search) — Level-by-level exploration; the go-to for shortest path in unweighted graphs. Problems: Shortest path in maze, word ladder, rotting oranges, open the lock.
🔴 DFS (Depth-First Search) — Go deep before backtracking; natural for exploring all paths or connected components. Problems: Number of islands, path sum, cycle detection, topological sort.
🔴 Topological Sort — Ordering nodes in a directed graph so dependencies come first. Problems: Course schedule, alien dictionary, build order.
🟡 Dijkstra's Algorithm — Shortest path in weighted graphs using a priority queue. Problems: Network delay time, cheapest flights within K stops.
🟡 Bellman-Ford — Shortest path that also handles negative edge weights. Problems: Cheapest flights within K stops (with constraints).

🟡 Good-to-Know:

Minimum Spanning Tree (Kruskal/Prim) — Connecting all nodes with minimum total edge weight. Problems: Min cost to connect all points.


Phase 4: Advanced Patterns (The "Hard" Problems)
This separates "hire" from "strong hire." DP alone accounts for ~15–20% of MAANG questions.
🔴 Must-Know:

Dynamic Programming (DP) — Solving problems by caching solutions to overlapping subproblems. This is a large category broken into sub-patterns:

1D DP — Single sequence decisions. Problems: Climbing stairs, house robber, coin change.
2D DP — Grid or two-sequence decisions. Problems: Longest common subsequence, edit distance, unique paths.
Knapsack Patterns — Choosing items under constraints to optimize value. Problems: 0/1 knapsack, partition equal subset sum, target sum.
Interval DP — Optimal decisions over ranges. Problems: Burst balloons, matrix chain multiplication.


Greedy Algorithms — Making the locally optimal choice at each step when that provably leads to a global optimum. Problems: Jump game, task scheduler, meeting rooms II, gas station.

🟡 Good-to-Know:

Segment Trees / Binary Indexed Trees — Efficient range queries and updates on arrays. Problems: Range sum query mutable, count of smaller numbers after self.
String Matching (KMP, Rabin-Karp) — Efficient substring search algorithms. Problems: Implement strStr, repeated substring pattern.
Math & Number Theory — GCD, primes, modular arithmetic for niche problems. Problems: Count primes, pow(x,n), happy number.


Phase 5: Interview-Specific Skills (The Meta-Game)
Knowing DSA isn't enough — you also need to perform in the interview.

🔴 Problem Decomposition — Breaking a vague problem into concrete steps before coding.
🔴 Brute Force → Optimize — Always start with the obvious O(n²) solution, then improve it; interviewers want to see your optimization thought process.
🔴 Edge Case Thinking — Empty inputs, single elements, overflow, duplicates — the things that separate clean solutions from buggy ones.
🔴 Communication — Talking through your thinking out loud so the interviewer can follow and help you.
🟡 Testing Your Code — Dry-running your solution with examples before saying "I'm done."


Suggested Practice Order (Priority Queue of Topics)
For maximum interview ROI, this is the order I'd recommend:

Big O → Arrays → Hash Maps → Two Pointers → Sliding Window
Binary Search → Sorting → Stacks → Queues
Linked Lists → Recursion → Binary Trees → BST
BFS → DFS → Backtracking → Topological Sort
Heaps → Tries → Greedy
Dynamic Programming (start with 1D, then 2D, then knapsack)
Union-Find → Graphs (advanced) → Remaining good-to-know topics


Where Do You Want to Start?
A few options based on your comfort level:

Phase 1 from the top — if you want a bulletproof foundation (recommended if you haven't done competitive coding before)
Big O + Arrays + Hash Maps — the highest-leverage starting trio
A specific topic you feel weakest in
Jump to a problem — and we'll work backward into the concepts

Also, one quick question — what language are you planning to code in during interviews? That way I can tailor examples and syntax when we start practicing.
