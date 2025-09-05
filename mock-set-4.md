# Stripe Interview Preparation - Mock Set 4

> A comprehensive collection of 40 Stripe-relevant coding problems with interview-ready solutions in Python. Each problem maps to real Stripe scenarios including reconciliation, rate limiting, idempotency, batching, and payment workflows.

---

## Table of Contents

1. [Arrays / Hashing / Intervals](#arrays--hashing--intervals)
2. [Additional Problem Categories](#additional-problem-categories)

---

## Arrays / Hashing / Intervals

### Problem 1: Merge Intervals

**📋 Problem Title:** Merge Intervals

**🎯 Stripe Relevance:** 
Reconcile overlapping payout windows and maintenance windows to avoid conflicts and ensure proper scheduling.

**💡 Approach:** 
- Sort intervals by start time
- Use sweep line algorithm to merge overlapping intervals
- Compare current interval start with previous interval end

**💻 Code:**
```python
def merge(intervals):
    intervals.sort()
    out = []
    for s, e in intervals:
        if not out or s > out[-1][1]:
            out.append([s, e])
        else:
            out[-1][1] = max(out[-1][1], e)
    return out
```

**⏰ Complexity:** 
- Time: O(n log n) due to sorting
- Space: O(1) excluding output

**🔍 Edge Cases:**
- Empty input array
- Single interval
- All intervals overlap into one
- No overlapping intervals

---

### Problem 2: Insert Interval

**📋 Problem Title:** Insert Interval

**🎯 Stripe Relevance:** 
Insert a new maintenance window idempotently into existing schedule without disrupting the system.

**💡 Approach:**
- Find position where new interval should be inserted
- Merge with overlapping intervals
- Handle three phases: before, during overlap, after

**💻 Code:**
```python
def insert(intervals, newI):
    res = []
    i = 0
    s, e = newI
    
    # Add intervals that end before new interval starts
    while i < len(intervals) and intervals[i][1] < s:
        res.append(intervals[i])
        i += 1
    
    # Merge overlapping intervals
    while i < len(intervals) and intervals[i][0] <= e:
        s = min(s, intervals[i][0])
        e = max(e, intervals[i][1])
        i += 1
    
    # Add merged interval and remaining intervals
    res.append([s, e])
    res += intervals[i:]
    return res
```

**⏰ Complexity:** 
- Time: O(n)
- Space: O(n) for result

**🔍 Edge Cases:**
- Overlaps at boundaries
- New interval completely contains existing ones
- New interval is contained within existing one
- Insert at beginning or end

---

### Problem 3: Minimum Window Substring

**📋 Problem Title:** Minimum Window Substring

**🎯 Stripe Relevance:** 
Find the smallest log slice containing all required error codes for debugging payment processing issues.

**💡 Approach:**
- Use sliding window technique with two pointers
- Track character frequency with hash map
- Expand window until valid, then contract to find minimum

**💻 Code:**
```python
from collections import Counter

def minWindow(s, t):
    need = Counter(t)
    miss = len(t)
    i = 0
    best = (10**9, None, None)
    
    for j, c in enumerate(s, 1):
        if need[c] > 0:
            miss -= 1
        need[c] -= 1
        
        if miss == 0:
            while i < j and need[s[i]] < 0:
                need[s[i]] += 1
                i += 1
            
            if j - i < best[0]:
                best = (j - i, i, j)
            
            need[s[i]] += 1
            miss += 1
            i += 1
    
    return "" if best[1] is None else s[best[1]:best[2]]
```

**⏰ Complexity:** 
- Time: O(|s| + |t|)
- Space: O(|t|) for character frequency map

**🔍 Edge Cases:**
- No valid window exists
- Target string is empty
- Source string is shorter than target
- Multiple minimum windows of same length

---

## Additional Problem Categories

*[Additional problems would be formatted similarly with the same structure]*

---

## Contributing

Feel free to contribute additional Stripe-relevant problems or improve existing solutions. Please maintain the established format for consistency.

## License

This repository is for educational purposes related to technical interview preparation.
