Awesome — here are 40 Stripe-relevant coding problems with concise, “interview-ready” solutions (Python), each mapping to real Stripe-ish scenarios like reconciliation, rate limiting, idempotency, batching, and graphy workflows. For each: when it’s relevant at Stripe, approach, clean code, and complexity.

Arrays / Hashing / Intervals
1) Merge Intervals

Stripe-why: reconcile overlapping payout windows, maintenance windows.
Approach: sort by start; sweep & merge.

def merge(intervals):
    intervals.sort()
    out=[]
    for s,e in intervals:
        if not out or s>out[-1][1]:
            out.append([s,e])
        else:
            out[-1][1]=max(out[-1][1],e)
    return out


Complexity: O(n log n). Edge: empty, single interval.

2) Insert Interval

Stripe-why: insert a new maintenance window idempotently.

def insert(intervals, newI):
    res=[]; i=0; s,e=newI
    while i<len(intervals) and intervals[i][1]<s: res.append(intervals[i]); i+=1
    while i<len(intervals) and intervals[i][0]<=e:
        s=min(s,intervals[i][0]); e=max(e,intervals[i][1]); i+=1
    res.append([s,e]); res+=intervals[i:]
    return res


Complexity: O(n). Edge: overlaps at boundaries.

3) Minimum Window Substring

Stripe-why: smallest log slice containing all error codes.

from collections import Counter
def minWindow(s,t):
    need=Counter(t); miss=len(t); i=0; best=(10**9,None,None)
    for j,c in enumerate(s,1):
        if need[c]>0: miss-=1
        need[c]-=1
        if miss==0:
            while i<j and need[s[i]]<0:
                need[s[i]]+=1; i+=1
            if j-i<best[0]: best=(j-i,i,j)
            need[s[i]]+=1; i+=1; miss+=1
    return "" if best[1] is None else s[best[1]:best[2]]


Complexity: O(n). Edge: no window.

4) Longest Substring Without Repeating

Stripe-why: longest streak of unique request IDs.

def lengthOfLongestSubstring(s):
    pos={}; i=ans=0
    for j,c in enumerate(s):
        if c in pos and pos[c]>=i: i=pos[c]+1
        pos[c]=j; ans=max(ans,j-i+1)
    return ans


Complexity: O(n).

5) Product of Array Except Self

Stripe-why: compute per-metric contribution without self.

def productExceptSelf(nums):
    n=len(nums); out=[1]*n
    p=1
    for i in range(n): out[i]=p; p*=nums[i]
    p=1
    for i in range(n-1,-1,-1): out[i]*=p; p*=nums[i]
    return out


Complexity: O(n), O(1) extra.

6) Top K Frequent Elements

Stripe-why: most frequent failure codes.

from collections import Counter
def topKFrequent(nums,k):
    return [x for x,_ in Counter(nums).most_common(k)]


Complexity: O(n log n) (or bucket sort O(n)).

7) Longest Consecutive Sequence

Stripe-why: longest consecutive days of payment success.

def longestConsecutive(nums):
    s=set(nums); best=0
    for x in s:
        if x-1 not in s:
            y=x
            while y in s: y+=1
            best=max(best,y-x)
    return best


Complexity: O(n).

8) Subarray Sum Equals K

Stripe-why: find a run of transactions summing to threshold.

from collections import defaultdict
def subarraySum(nums,k):
    pref=0; seen=defaultdict(int,{0:1}); ans=0
    for v in nums:
        pref+=v
        ans+=seen[pref-k]
        seen[pref]+=1
    return ans


Complexity: O(n).

9) 3Sum

Stripe-why: detect offsetting entries that net to zero.

def threeSum(a):
    a.sort(); n=len(a); res=[]
    for i in range(n-2):
        if i and a[i]==a[i-1]: continue
        l,r=i+1,n-1
        while l<r:
            s=a[i]+a[l]+a[r]
            if s<0: l+=1
            elif s>0: r-=1
            else:
                res.append([a[i],a[l],a[r]])
                l+=1; r-=1
                while l<r and a[l]==a[l-1]: l+=1
                while l<r and a[r]==a[r+1]: r-=1
    return res


Complexity: O(n²).

10) Trapping Rain Water

Stripe-why: capacity planning (peaks/valleys metaphor).

def trap(h):
    l=0; r=len(h)-1; L=R=0; ans=0
    while l<r:
        if h[l]<h[r]:
            L=max(L,h[l]); ans+=L-h[l]; l+=1
        else:
            R=max(R,h[r]); ans+=R-h[r]; r-=1
    return ans


Complexity: O(n).

11) Interval Scheduling (Non-overlapping)

Stripe-why: choose non-overlapping payouts/maintenance.

def eraseOverlapIntervals(itv):
    itv.sort(key=lambda x:x[1])
    end=-10**18; keep=0
    for s,e in itv:
        if s>=end: keep+=1; end=e
    return len(itv)-keep


Complexity: O(n log n).

12) Meeting Rooms II (Min Rooms)

Stripe-why: min shards/queues to process concurrent jobs.

import heapq
def minMeetingRooms(itv):
    itv.sort()
    pq=[]
    for s,e in itv:
        if pq and pq[0]<=s: heapq.heapreplace(pq,e)
        else: heapq.heappush(pq,e)
    return len(pq)


Complexity: O(n log n).

Stacks / Queues / Heaps
13) Valid Parentheses

Stripe-why: validate expressions/config strings.

def isValid(s):
    m={')':'(',']':'[','}':'{'}; st=[]
    for c in s:
        if c in m:
            if not st or st.pop()!=m[c]: return False
        else: st.append(c)
    return not st

14) Min Stack

Stripe-why: keep min latency/amount alongside pushes.

class MinStack:
    def __init__(self): self.st=[]
    def push(self,x): self.st.append((x, x if not self.st else min(x,self.st[-1][1])))
    def pop(self): self.st.pop()
    def top(self): return self.st[-1][0]
    def getMin(self): return self.st[-1][1]

15) Kth Largest in Stream

Stripe-why: p99 latency tracking; keep top-k.

import heapq
class KthLargest:
    def __init__(self,k,nums):
        self.k=k; self.h=nums; heapq.heapify(self.h)
        while len(self.h)>k: heapq.heappop(self.h)
    def add(self,val):
        if len(self.h)<self.k: heapq.heappush(self.h,val)
        elif val>self.h[0]: heapq.heapreplace(self.h,val)
        return self.h[0]

16) Sliding Window Maximum

Stripe-why: rolling max throughput/latency.

from collections import deque
def maxSlidingWindow(nums,k):
    dq=deque(); res=[]
    for i,x in enumerate(nums):
        while dq and dq[0]<=i-k: dq.popleft()
        while dq and nums[dq[-1]]<=x: dq.pop()
        dq.append(i)
        if i>=k-1: res.append(nums[dq[0]])
    return res

17) Evaluate Reverse Polish Notation

Stripe-why: simple expression evaluators in config.

def evalRPN(tokens):
    st=[]
    for t in tokens:
        if t in '+-*/':
            b=st.pop(); a=st.pop()
            st.append(int(a/b) if t=='/' else a+b if t=='+' else a-b if t=='-' else a*b)
        else: st.append(int(t))
    return st[-1]

18) LRU Cache (Design)

Stripe-why: cache merchant config; avoid thundering herds.

from collections import OrderedDict
class LRUCache:
    def __init__(self,cap): self.cap=cap; self.od=OrderedDict()
    def get(self,k):
        if k not in self.od: return -1
        self.od.move_to_end(k); return self.od[k]
    def put(self,k,v):
        if k in self.od: self.od.move_to_end(k)
        self.od[k]=v
        if len(self.od)>self.cap: self.od.popitem(last=False)

Linked Lists
19) Reverse Linked List
def reverseList(head):
    prev=None; cur=head
    while cur:
        nxt=cur.next; cur.next=prev; prev=cur; cur=nxt
    return prev

20) Add Two Numbers

Stripe-why: sum amounts digit-by-digit (minor units).

def addTwoNumbers(l1,l2):
    carry=0; dummy=ListNode(0); cur=dummy
    while l1 or l2 or carry:
        s=(l1.val if l1 else 0)+(l2.val if l2 else 0)+carry
        carry, d = divmod(s,10)
        cur.next=ListNode(d); cur=cur.next
        l1=l1.next if l1 else None; l2=l2.next if l2 else None
    return dummy.next

21) Detect Cycle (start)
def detectCycle(head):
    slow=fast=head
    while fast and fast.next:
        slow=slow.next; fast=fast.next.next
        if slow==fast:
            slow=head
            while slow!=fast:
                slow=slow.next; fast=fast.next
            return slow
    return None

22) Copy List with Random Pointer
def copyRandomList(head):
    if not head: return None
    cur=head
    while cur:
        nxt=cur.next; cur.next=Node(cur.val,nxt); cur=nxt
    cur=head
    while cur:
        if cur.random: cur.next.random=cur.random.next
        cur=cur.next.next
    cur=head; new=head.next; copy=new
    while cur:
        cur.next=cur.next.next
        copy.next=copy.next.next if copy.next else None
        cur=cur.next; copy=copy.next
    return new

Trees / BST
23) Validate BST

Stripe-why: invariant checking on configs.

def isValidBST(root):
    def dfs(n,lo,hi):
        if not n: return True
        if not (lo<n.val<hi): return False
        return dfs(n.left,lo,n.val) and dfs(n.right,n.val,hi)
    return dfs(root,float('-inf'),float('inf'))

24) Kth Smallest in BST
def kthSmallest(root,k):
    st=[]; cur=root
    while True:
        while cur: st.append(cur); cur=cur.left
        cur=st.pop(); k-=1
        if k==0: return cur.val
        cur=cur.right

25) Serialize/Deserialize Binary Tree
def serialize(root):
    res=[]
    def dfs(n):
        if not n: res.append('#'); return
        res.append(str(n.val)); dfs(n.left); dfs(n.right)
    dfs(root); return ' '.join(res)

def deserialize(s):
    it=iter(s.split())
    def build():
        v=next(it)
        if v=='#': return None
        n=TreeNode(int(v)); n.left=build(); n.right=build(); return n
    return build()

26) Lowest Common Ancestor (Binary Tree)
def lowestCommonAncestor(root,p,q):
    if not root or root==p or root==q: return root
    L=lowestCommonAncestor(root.left,p,q)
    R=lowestCommonAncestor(root.right,p,q)
    return root if L and R else L or R

27) Diameter of Binary Tree
def diameterOfBinaryTree(root):
    ans=0
    def dfs(n):
        nonlocal ans
        if not n: return 0
        L=dfs(n.left); R=dfs(n.right)
        ans=max(ans,L+R)
        return 1+max(L,R)
    dfs(root); return ans

Graphs / BFS / Toposort
28) Number of Islands

Stripe-why: count disconnected failure regions/clusters.

def numIslands(g):
    if not g: return 0
    m,n=len(g),len(g[0]); ans=0
    def dfs(i,j):
        if i<0 or j<0 or i>=m or j>=n or g[i][j]!='1': return
        g[i][j]='0'
        for di,dj in [(1,0),(-1,0),(0,1),(0,-1)]: dfs(i+di,j+dj)
    for i in range(m):
        for j in range(n):
            if g[i][j]=='1': ans+=1; dfs(i,j)
    return ans

29) Clone Graph
def cloneGraph(node):
    if not node: return None
    mp={}
    def dfs(n):
        if n in mp: return mp[n]
        copy=Node(n.val,[])
        mp[n]=copy
        for nb in n.neighbors: copy.neighbors.append(dfs(nb))
        return copy
    return dfs(node)

30) Course Schedule (Cycle detection / Toposort)

Stripe-why: dependency order for migrations/jobs.

from collections import defaultdict,deque
def canFinish(n,prereq):
    indeg=[0]*n; g=defaultdict(list)
    for a,b in prereq: g[b].append(a); indeg[a]+=1
    q=deque([i for i in range(n) if indeg[i]==0]); seen=0
    while q:
        u=q.popleft(); seen+=1
        for v in g[u]:
            indeg[v]-=1
            if indeg[v]==0: q.append(v)
    return seen==n

31) Course Schedule II (Order)
def findOrder(n,prereq):
    from collections import defaultdict,deque
    indeg=[0]*n; g=defaultdict(list)
    for a,b in prereq: g[b].append(a); indeg[a]+=1
    q=deque([i for i in range(n) if indeg[i]==0]); out=[]
    while q:
        u=q.popleft(); out.append(u)
        for v in g[u]:
            indeg[v]-=1
            if indeg[v]==0: q.append(v)
    return out if len(out)==n else []

32) Word Ladder (Shortest transformation)

Stripe-why: minimum steps to migrate schema A→B (state transitions).

from collections import deque
def ladderLength(begin,end,wordList):
    s=set(wordList)
    if end not in s: return 0
    q=deque([(begin,1)])
    while q:
        w,d=q.popleft()
        if w==end: return d
        arr=list(w)
        for i,ch in enumerate(arr):
            orig=arr[i]
            for c in 'abcdefghijklmnopqrstuvwxyz':
                if c==orig: continue
                arr[i]=c; nxt=''.join(arr)
                if nxt in s:
                    s.remove(nxt); q.append((nxt,d+1))
            arr[i]=orig
    return 0

33) Pacific Atlantic Water Flow

Stripe-why: reachability from multiple sources (multi-region failover).

def pacificAtlantic(h):
    if not h: return []
    m,n=len(h),len(h[0])
    def bfs(starts):
        from collections import deque
        vis=set(starts); q=deque(starts)
        while q:
            i,j=q.popleft()
            for di,dj in [(1,0),(-1,0),(0,1),(0,-1)]:
                x,y=i+di,j+dj
                if 0<=x<m and 0<=y<n and (x,y) not in vis and h[x][y]>=h[i][j]:
                    vis.add((x,y)); q.append((x,y))
        return vis
    pac=bfs([(0,j) for j in range(n)]+[(i,0) for i in range(m)])
    atl=bfs([(m-1,j) for j in range(n)]+[(i,n-1) for i in range(m)])
    return list(pac & atl)

34) Rotting Oranges (Multi-source BFS)

Stripe-why: propagation delay (incident spread).

from collections import deque
def orangesRotting(g):
    m,n=len(g),len(g[0]); q=deque(); fresh=0
    for i in range(m):
        for j in range(n):
            if g[i][j]==2: q.append((i,j,0))
            elif g[i][j]==1: fresh+=1
    t=0
    while q:
        i,j,d=q.popleft(); t=max(t,d)
        for di,dj in [(1,0),(-1,0),(0,1),(0,-1)]:
            x,y=i+di,j+dj
            if 0<=x<m and 0<=y<n and g[x][y]==1:
                g[x][y]=2; fresh-=1; q.append((x,y,d+1))
    return -1 if fresh else t

Dynamic Programming / Greedy
35) Coin Change

Stripe-why: minimum refunds to reach target; denomination logic.

def coinChange(coins,amt):
    INF=10**9; dp=[0]+[INF]*amt
    for a in range(1,amt+1):
        for c in coins:
            if a>=c: dp[a]=min(dp[a],dp[a-c]+1)
    return -1 if dp[amt]==INF else dp[amt]

36) Longest Increasing Subsequence (O(n log n))

Stripe-why: longest improving KPIs; scheduling chains.

import bisect
def lengthOfLIS(nums):
    tails=[]
    for x in nums:
        i=bisect.bisect_left(tails,x)
        if i==len(tails): tails.append(x)
        else: tails[i]=x
    return len(tails)

37) Edit Distance

Stripe-why: diffing configs safely.

def minDistance(a,b):
    m,n=len(a),len(b)
    dp=[[0]*(n+1) for _ in range(m+1)]
    for i in range(m+1): dp[i][0]=i
    for j in range(n+1): dp[0][j]=j
    for i in range(1,m+1):
        for j in range(1,n+1):
            dp[i][j]=dp[i-1][j-1] if a[i-1]==b[j-1] else 1+min(dp[i-1][j],dp[i][j-1],dp[i-1][j-1])
    return dp[m][n]

38) Word Break

Stripe-why: compose features from dictionary of flags.

def wordBreak(s,wd):
    wd=set(wd); n=len(s); dp=[False]*(n+1); dp[0]=True
    for i in range(1,n+1):
        for j in range(i):
            if dp[j] and s[j:i] in wd: dp[i]=True; break
    return dp[-1]

39) Partition Equal Subset Sum

Stripe-why: split payouts evenly across rails.

def canPartition(nums):
    s=sum(nums)
    if s%2: return False
    target=s//2
    dp=set([0])
    for x in nums:
        dp |= {v+x for v in list(dp) if v+x<=target}
    return target in dp

40) Maximal Square

Stripe-why: densest contiguous healthy region.

def maximalSquare(mat):
    if not mat: return 0
    m,n=len(mat),len(mat[0]); dp=[0]*(n+1); best=0
    for i in range(1,m+1):
        prev=0
        for j in range(1,n+1):
            temp=dp[j]
            if mat[i-1][j-1]=='1' or mat[i-1][j-1]==1:
                dp[j]=min(dp[j],dp[j-1],prev)+1
                best=max(best,dp[j])
            else: dp[j]=0
            prev=temp
    return best*best

Extra Stripe-flavored patterns
41) (Bonus) Reconcile Two Ledgers (Symmetric diff by id)

Stripe-why: unmatched transactions.

def reconcile(stripe,bank):  # lists of dicts with 'id'
    m={t['id']:t for t in stripe}; unmatched=[]; discrep=[]
    for b in bank:
        s=m.pop(b['id'],None)
        if not s: unmatched.append(b)
        elif s!=b: discrep.append((s,b))
    unmatched += list(m.values())
    return unmatched, discrep

42) (Bonus) Rate Limit (Sliding window counters)
from collections import deque, defaultdict
class RateLimiter:
    def __init__(self, limit, window): self.L=limit; self.W=window; self.q=defaultdict(deque)
    def allow(self, key, now):
        q=self.q[key]
        while q and now-q[0]>=self.W: q.popleft()
        if len(q)<self.L: q.append(now); return True
        return False


(Use these two as practice extras; the 40 primary above already cover LeetCode-style breadth.)

How to drill effectively

For each: explain brute force → optimal, note time/space, and test edge cases (empty, large, duplicates, negative, boundaries).

Use integer minor units for money, never floats.

Verbalize invariants (e.g., “window contains all required counts”, “heap holds current ends”, “idempotency: same input → same output”).
