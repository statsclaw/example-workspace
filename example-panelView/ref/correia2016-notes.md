# Correia (2016) — Key Concepts for panelView Network Visualization

## Reference
Correia, Sergio. 2016. "A Feasible Estimator for Linear Models with Multi-Way Fixed Effects." Duke University. Preliminary Version. https://scorreia.com/research/hdfe.pdf

## Figure 1: Bipartite Graph of Fixed Effects
- Individuals = red circles, Firms = blue squares
- Edges = observation (individual i worked at firm j)
- Edge weights = number of observations where the pair co-occurs
- The graph IS the structure of the D'D matrix (normal equations)

## Figure 2: CEO-Firm Connection Graph
- Shows a larger bipartite graph of CEO-Firm connections
- Two node shapes (circles vs squares) for the two fixed effect dimensions
- Layout reveals connected components and isolated clusters
- **This is the visual we want to replicate in panelView** but with unit × time (and optionally more FE dimensions)

## Section 3.4: Graph Pruning — Singletons (Degree-1 Nodes)
- "If a firm only had one CEO through the sample, then its associated vertex has a degree of 1"
- Degree-1 vertices can be removed: the normal equation has a triangular structure for them
- After removing degree-1, proceed to degree-2 (path compression)
- This is the "Greedy Elimination" algorithm of Koutis, Miller, and Peng (2010)
- **For panelView**: identify units observed in only 1 period OR periods with only 1 unit. These are singletons that can cause identification problems in FE estimation.

## Section 3.5: Connected Components (RCM Algorithm)
- The reverse Cuthill-McKee algorithm reorders the Laplacian and reveals disconnected subgraphs
- Each disconnected subgraph = a connected component
- Fixed effects are only identified WITHIN a connected component
- Units in different components cannot be compared
- **For panelView**: draw convex hulls around connected components to show which groups of units/periods are connected

## Section 4.1: k-partite Extension (f > 2 fixed effects)
- For f = 3+ (e.g., individual × firm × time), the MAP framework iterates across all FE dimensions
- The Laplacian solver can be embedded: combine any two sets of projections into a joint projection, iterate with remaining
- Graph becomes k-partite: k node types, edges only between different types
- **For panelView**: support additional FE columns beyond unit + time. Each FE dimension = a different node shape/color.

## Key Graph-Theoretic Definitions (Section 3.1)
- Weighted undirected graph G = (V, E, w)
- V = set of vertices (fixed effect categories)
- E = set of edges (pairs of FE categories sharing observations)
- w(e) > 0 = edge weight (number of shared observations)
- Laplacian L: L[a,b] = w(a) if a=b, -w(a,b) if (a,b) in E, 0 otherwise
- L = B'R^{-1}B where B is incidence matrix, R is resistance matrix

## Relevance to Panel Data
In a standard TSCS panel with unit and time FE:
- Units = one node set (N nodes)
- Time periods = another node set (T nodes)
- Edge (i, t) exists iff unit i is observed at time t
- Edge weight = 1 (or number of obs if repeated measures)
- Balanced panel → complete bipartite graph K_{N,T}
- Unbalanced panel → sparse bipartite graph with potentially disconnected components and singletons
