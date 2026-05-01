# ALGO-CORE — ICPC Algorithm Engine v4.0

> Complete algorithm template library. Pattern recognition → strategy → instantiation → verification.
> All templates: O-optimal, overflow-safe, edge-case-hardened, competition-ready.

---

## Pattern Recognition Engine

### Step 1: Classify Problem Type

```
Read constraints → identify N, Q, time limit, memory limit
Read objective → identify operation type

N ≤ 20          → bitmask DP or brute force acceptable
N ≤ 500         → O(N³) acceptable (Floyd-Warshall, cubic DP)
N ≤ 5,000       → O(N²) acceptable
N ≤ 300,000     → O(N log N) required
N ≤ 10^6        → O(N) or O(N log N)
N ≤ 10^18       → O(log N) or O(√N)

Q queries + N   → precompute O(N) or O(N log N), answer O(1) or O(log N) each
```

### Step 2: Pattern → Algorithm Map

```
GRAPH PATTERNS:
  "shortest path, weighted"           → Dijkstra (ALGO-01)
  "shortest path, unweighted"         → BFS (ALGO-02)
  "all-pairs shortest path"           → Floyd-Warshall (ALGO-03)
  "minimum spanning tree"             → Kruskal + DSU (ALGO-04)
  "max flow / min cut"                → Dinic's (ALGO-12)
  "bipartite matching"                → Hopcroft-Karp (ALGO-13)
  "SCC / strongly connected"          → Tarjan's (ALGO-14)
  "bridge / articulation point"       → DFS lowlink (ALGO-14 variant)
  "tree path queries"                 → HLD (ALGO-18)
  "tree distance / LCA"               → Binary lifting O(log N)
  "divide on tree"                    → Centroid decomp (ALGO-19)

DP PATTERNS:
  "subset sum / knapsack"             → 0-1 DP (ALGO-05)
  "longest common subsequence"        → LCS DP (ALGO-06)
  "longest increasing subsequence"    → DP + BinSearch (ALGO-07)
  "interval DP"                       → O(N³) DP on intervals
  "bitmask DP / TSP"                  → O(2^N × N) DP
  "digit DP"                          → O(N × states) on digits
  "matrix exponentiation"             → O(K³ log N) for linear recurrence

STRING PATTERNS:
  "find pattern in text"              → KMP (ALGO-09)
  "multiple pattern search"           → Aho-Corasick (ALGO-10)
  "suffix array / LCP"                → SA-IS (ALGO-11)
  "palindrome detection"              → Manacher's O(N)
  "string hashing"                    → Polynomial rolling hash

RANGE QUERY PATTERNS:
  "range sum / min / max"             → Segment Tree (ALGO-15)
  "range update + range query"        → Lazy Segment Tree (ALGO-16)
  "point update + prefix sum"         → BIT / Fenwick Tree (ALGO-17)
  "offline range queries"             → Mo's Algorithm (ALGO-24)

MATH PATTERNS:
  "polynomial multiplication"         → FFT / NTT (ALGO-20)
  "nCr mod p, factorials"             → Precompute factorials (ALGO-21)
  "GCD / LCM"                         → Euclidean algorithm
  "modular inverse"                   → Fermat's little theorem (p prime)
  "primality / factorization"         → Sieve of Eratosthenes O(N log log N)
  "discrete log"                      → Baby-step giant-step O(√P)
  "Chinese remainder theorem"         → CRT extended GCD

GEOMETRY PATTERNS:
  "convex hull"                       → Graham scan / Andrew's monotone (ALGO-23)
  "point in polygon"                  → Ray casting
  "line intersection"                 → Cross product
  "closest pair of points"            → Divide and conquer O(N log N)

GREEDY PATTERNS:
  "interval scheduling"               → Sort by end time (ALGO-08)
  "interval covering"                 → Sort by start, extend right
  "task scheduling (deadlines)"       → Earliest deadline first
  "activity selection"                → Greedy by finish time

GAME THEORY:
  "Nim / impartial games"             → Sprague-Grundy (ALGO-25)
  "combinatorial game"                → Grundy values / mex
```

---

## Algorithm Templates

### ALGO-01: Dijkstra (Shortest Path, Weighted)

```cpp
#include <bits/stdc++.h>
using namespace std;
typedef long long ll;
typedef pair<ll,int> pli;

const ll INF = 1e18;

// Adjacency list: adj[u] = {(weight, v), ...}
vector<pair<ll,int>> adj[MAXN];

vector<ll> dijkstra(int src, int n) {
    vector<ll> dist(n + 1, INF);
    priority_queue<pli, vector<pli>, greater<pli>> pq;
    dist[src] = 0;
    pq.push({0, src});
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;   // stale entry
        for (auto [w, v] : adj[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.push({dist[v], v});
            }
        }
    }
    return dist; // dist[i] = shortest path src → i
}
// Complexity: O((V+E) log V)
// Precondition: non-negative edge weights
```

### ALGO-02: BFS (Shortest Path, Unweighted)

```cpp
vector<int> bfs(int src, int n, vector<vector<int>>& adj) {
    vector<int> dist(n + 1, -1);
    queue<int> q;
    dist[src] = 0;
    q.push(src);
    while (!q.empty()) {
        int u = q.front(); q.pop();
        for (int v : adj[u]) {
            if (dist[v] == -1) {
                dist[v] = dist[u] + 1;
                q.push(v);
            }
        }
    }
    return dist; // dist[i] = -1 if unreachable
}
// Complexity: O(V+E)
```

### ALGO-03: Floyd-Warshall (All-Pairs Shortest Path)

```cpp
// dist[i][j] = shortest path i → j
void floyd(int n, vector<vector<ll>>& dist) {
    // Initialize: dist[i][i]=0, dist[i][j]=edge or INF
    for (int k = 1; k <= n; k++)
        for (int i = 1; i <= n; i++)
            for (int j = 1; j <= n; j++)
                if (dist[i][k] < INF && dist[k][j] < INF)
                    dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j]);
    // Negative cycle: dist[i][i] < 0 for some i
}
// Complexity: O(V³)  — use only when V ≤ 500
```

### ALGO-04: Kruskal + DSU (Minimum Spanning Tree)

```cpp
struct DSU {
    vector<int> p, rank_;
    DSU(int n) : p(n+1), rank_(n+1, 0) { iota(p.begin(), p.end(), 0); }
    int find(int x) { return p[x] == x ? x : p[x] = find(p[x]); }
    bool unite(int a, int b) {
        a = find(a); b = find(b);
        if (a == b) return false;
        if (rank_[a] < rank_[b]) swap(a, b);
        p[b] = a;
        if (rank_[a] == rank_[b]) rank_[a]++;
        return true;
    }
};

ll kruskal(int n, vector<tuple<ll,int,int>>& edges) {
    sort(edges.begin(), edges.end()); // sort by weight
    DSU dsu(n);
    ll mst_weight = 0;
    int edges_added = 0;
    for (auto [w, u, v] : edges) {
        if (dsu.unite(u, v)) {
            mst_weight += w;
            if (++edges_added == n - 1) break;
        }
    }
    return (edges_added == n - 1) ? mst_weight : -1; // -1 = disconnected
}
// Complexity: O(E log E)
```

### ALGO-05: 0-1 Knapsack DP

```cpp
// items: {weight, value}, capacity W
// Returns max value achievable within capacity W
ll knapsack(int n, int W, vector<int>& wt, vector<ll>& val) {
    vector<ll> dp(W + 1, 0);
    for (int i = 0; i < n; i++)
        for (int w = W; w >= wt[i]; w--)  // reverse to prevent reuse
            dp[w] = max(dp[w], dp[w - wt[i]] + val[i]);
    return dp[W];
}
// Complexity: O(N×W)
// For unbounded knapsack: iterate w forward (not reverse)
```

### ALGO-06: LCS (Longest Common Subsequence)

```cpp
int lcs(string& a, string& b) {
    int n = a.size(), m = b.size();
    vector<vector<int>> dp(n+1, vector<int>(m+1, 0));
    for (int i = 1; i <= n; i++)
        for (int j = 1; j <= m; j++)
            dp[i][j] = (a[i-1] == b[j-1])
                ? dp[i-1][j-1] + 1
                : max(dp[i-1][j], dp[i][j-1]);
    return dp[n][m];
}
// Complexity: O(N×M)
// Space optimization: use 2 rows → O(min(N,M))
```

### ALGO-07: LIS (Longest Increasing Subsequence) O(N log N)

```cpp
int lis(vector<int>& a) {
    vector<int> tails; // tails[i] = smallest tail of IS of length i+1
    for (int x : a) {
        auto it = lower_bound(tails.begin(), tails.end(), x);
        if (it == tails.end()) tails.push_back(x);
        else *it = x;
    }
    return tails.size();
}
// For non-strictly increasing: replace lower_bound with upper_bound
// Complexity: O(N log N)
```

### ALGO-08: Interval Scheduling (Greedy)

```cpp
// Returns max non-overlapping intervals
int scheduleIntervals(vector<pair<int,int>>& intervals) {
    sort(intervals.begin(), intervals.end(),
         [](auto& a, auto& b){ return a.second < b.second; }); // sort by end
    int count = 0, last_end = INT_MIN;
    for (auto [s, e] : intervals) {
        if (s >= last_end) { // non-overlapping
            count++;
            last_end = e;
        }
    }
    return count;
}
// Complexity: O(N log N)
```

### ALGO-09: KMP (String Pattern Matching)

```cpp
vector<int> buildKMP(string& p) {
    int m = p.size();
    vector<int> fail(m, 0);
    for (int i = 1; i < m; i++) {
        int j = fail[i-1];
        while (j > 0 && p[i] != p[j]) j = fail[j-1];
        if (p[i] == p[j]) j++;
        fail[i] = j;
    }
    return fail;
}

vector<int> kmpSearch(string& text, string& pattern) {
    auto fail = buildKMP(pattern);
    vector<int> matches;
    int j = 0;
    for (int i = 0; i < (int)text.size(); i++) {
        while (j > 0 && text[i] != pattern[j]) j = fail[j-1];
        if (text[i] == pattern[j]) j++;
        if (j == (int)pattern.size()) {
            matches.push_back(i - j + 1); // 0-indexed match start
            j = fail[j-1];
        }
    }
    return matches;
}
// Complexity: O(N+M)
```

### ALGO-12: Dinic's (Max Flow)

```cpp
struct Dinic {
    struct Edge { int to, rev; ll cap; };
    vector<vector<Edge>> g;
    vector<int> level, iter;
    int n;
    Dinic(int n) : n(n), g(n), level(n), iter(n) {}

    void addEdge(int u, int v, ll cap) {
        g[u].push_back({v, (int)g[v].size(), cap});
        g[v].push_back({u, (int)g[u].size()-1, 0});
    }

    bool bfs(int s, int t) {
        fill(level.begin(), level.end(), -1);
        queue<int> q;
        level[s] = 0; q.push(s);
        while (!q.empty()) {
            int v = q.front(); q.pop();
            for (auto& e : g[v])
                if (e.cap > 0 && level[e.to] < 0) {
                    level[e.to] = level[v] + 1;
                    q.push(e.to);
                }
        }
        return level[t] >= 0;
    }

    ll dfs(int v, int t, ll f) {
        if (v == t) return f;
        for (int& i = iter[v]; i < (int)g[v].size(); i++) {
            Edge& e = g[v][i];
            if (e.cap > 0 && level[v] < level[e.to]) {
                ll d = dfs(e.to, t, min(f, e.cap));
                if (d > 0) { e.cap -= d; g[e.to][e.rev].cap += d; return d; }
            }
        }
        return 0;
    }

    ll maxflow(int s, int t) {
        ll flow = 0;
        while (bfs(s, t)) {
            fill(iter.begin(), iter.end(), 0);
            ll d;
            while ((d = dfs(s, t, 1e18)) > 0) flow += d;
        }
        return flow;
    }
};
// Complexity: O(V²E)  general | O(E√V) unit graphs
```

### ALGO-15: Segment Tree (Range Query)

```cpp
// Generic segment tree — customize combine() for min/max/sum/gcd
struct SegTree {
    int n;
    vector<ll> tree;
    SegTree(int n, ll init = 0) : n(n), tree(4*n, init) {}

    ll combine(ll a, ll b) { return a + b; } // change for min/max/gcd

    void build(vector<ll>& a, int node, int l, int r) {
        if (l == r) { tree[node] = a[l]; return; }
        int mid = (l + r) / 2;
        build(a, 2*node, l, mid);
        build(a, 2*node+1, mid+1, r);
        tree[node] = combine(tree[2*node], tree[2*node+1]);
    }
    void build(vector<ll>& a) { build(a, 1, 0, n-1); }

    void update(int node, int l, int r, int pos, ll val) {
        if (l == r) { tree[node] = val; return; }
        int mid = (l + r) / 2;
        if (pos <= mid) update(2*node, l, mid, pos, val);
        else update(2*node+1, mid+1, r, pos, val);
        tree[node] = combine(tree[2*node], tree[2*node+1]);
    }
    void update(int pos, ll val) { update(1, 0, n-1, pos, val); }

    ll query(int node, int l, int r, int ql, int qr) {
        if (qr < l || r < ql) return 0; // identity for combine (0 for sum, INF for min)
        if (ql <= l && r <= qr) return tree[node];
        int mid = (l + r) / 2;
        return combine(query(2*node, l, mid, ql, qr),
                       query(2*node+1, mid+1, r, ql, qr));
    }
    ll query(int l, int r) { return query(1, 0, n-1, l, r); }
};
// Complexity: O(log N) per update/query | O(N) build
```

### ALGO-17: Binary Indexed Tree / Fenwick

```cpp
struct BIT {
    int n;
    vector<ll> tree;
    BIT(int n) : n(n), tree(n+1, 0) {}

    void update(int i, ll delta) {
        for (++i; i <= n; i += i & -i) tree[i] += delta;
    }

    ll query(int i) { // prefix sum [0..i]
        ll s = 0;
        for (++i; i > 0; i -= i & -i) s += tree[i];
        return s;
    }

    ll query(int l, int r) { return query(r) - (l ? query(l-1) : 0); }
};
// Complexity: O(log N) per update/query | O(N) build
```

### ALGO-21: Math Templates (Modular Arithmetic)

```cpp
const ll MOD = 1e9 + 7; // or 998244353 for NTT

ll pw(ll base, ll exp, ll mod = MOD) {
    ll result = 1; base %= mod;
    for (; exp > 0; exp >>= 1) {
        if (exp & 1) result = result * base % mod;
        base = base * base % mod;
    }
    return result;
}

ll inv(ll a, ll mod = MOD) { return pw(a, mod - 2, mod); } // mod must be prime

// Precompute factorials for nCr
const int MAXN = 2e6 + 5;
ll fact[MAXN], inv_fact[MAXN];
void precompute() {
    fact[0] = 1;
    for (int i = 1; i < MAXN; i++) fact[i] = fact[i-1] * i % MOD;
    inv_fact[MAXN-1] = inv(fact[MAXN-1]);
    for (int i = MAXN-2; i >= 0; i--) inv_fact[i] = inv_fact[i+1] * (i+1) % MOD;
}

ll nCr(int n, int r) {
    if (r < 0 || r > n) return 0;
    return fact[n] % MOD * inv_fact[r] % MOD * inv_fact[n-r] % MOD;
}

// GCD / LCM
ll gcd(ll a, ll b) { return b ? gcd(b, a%b) : a; }
ll lcm(ll a, ll b) { return a / gcd(a,b) * b; } // divide first to prevent overflow
```

### ALGO-22: Trie

```cpp
struct Trie {
    int ch[MAXN][26], cnt[MAXN], end_[MAXN], tot = 1;

    void insert(string& s) {
        int cur = 1;
        for (char c : s) {
            int x = c - 'a';
            if (!ch[cur][x]) ch[cur][x] = ++tot;
            cur = ch[cur][x];
            cnt[cur]++;
        }
        end_[cur]++;
    }

    int search(string& s) { // returns count of strings with this prefix
        int cur = 1;
        for (char c : s) {
            int x = c - 'a';
            if (!ch[cur][x]) return 0;
            cur = ch[cur][x];
        }
        return cnt[cur];
    }
};
// For XOR trie: use 30 bits, branch on bit value
```

---

## Complexity Verification Matrix

```
Given N=10^5, time limit=1s:
  O(N)        → ~10^5 ops   ✓ fast
  O(N log N)  → ~1.7×10^6  ✓ fast
  O(N √N)     → ~3.2×10^7  ✓ acceptable
  O(N^1.5)    → same
  O(N²)       → 10^10       ✗ TLE

Given N=10^3, time limit=1s:
  O(N²)       → 10^6        ✓ fast
  O(N³)       → 10^9        ⚠ marginal (tight)
  O(2^N)      → 10^300      ✗ TLE

Given N=10^6, time limit=2s:
  O(N)        → 10^6        ✓ fast
  O(N log N)  → 2×10^7      ✓ fast
  O(N²)       → 10^12       ✗ TLE
```

---

## Edge Case Checklist (R12)

Before emitting any solution, verify:
```
□ Empty input (N=0, empty array/string)
□ Single element (N=1)
□ All elements equal
□ Maximum constraints (N=10^6 or problem max)
□ Adversarial / worst case (sorted desc for sort-based, star graph for tree)
□ Integer overflow: sums can exceed 10^9 → use long long
□ Negative values: does algorithm assume non-negative?
□ Disconnected graph: is the graph always connected?
□ Self-loops / multi-edges: does problem allow them?
□ 0-indexing vs 1-indexing: consistent throughout
□ Modular arithmetic: applied at every intermediate step?
```

---

## Competitive Boilerplate

```cpp
#include <bits/stdc++.h>
using namespace std;
typedef long long ll;
typedef pair<int,int> pii;
typedef pair<ll,ll> pll;
typedef vector<int> vi;
typedef vector<ll> vl;
typedef vector<pii> vpii;

const ll MOD = 1e9 + 7;
const ll INF = 1e18;
const int MAXN = 2e5 + 5;

#define all(x) (x).begin(), (x).end()
#define sz(x) (int)(x).size()
#define pb push_back
#define fi first
#define se second

void solve() {
    // solution here
}

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(NULL);

    int T = 1;
    // cin >> T; // uncomment for multiple test cases
    while (T--) solve();
    return 0;
}
```

---

## Auto-Solve Protocol

When given a competitive programming problem:

```
1. READ constraints → determine N, time limit
2. IDENTIFY pattern → match to Pattern Recognition Engine
3. SELECT template → pick ALGO-XX from registry
4. CHECK complexity → verify O(?) fits constraints (R11)
5. INSTANTIATE → fill in problem-specific parts
6. VERIFY edges → run R12 checklist mentally
7. EMIT solution with:
   {
     "algorithm": "ALGO-XX",
     "complexity_time": "O(?)",
     "complexity_space": "O(?)",
     "fits_constraints": true,
     "edge_cases_verified": ["...", "..."],
     "overflow_safe": true,
     "solution": "<code>",
     "confidence": N
   }
```
