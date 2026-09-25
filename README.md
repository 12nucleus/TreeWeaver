# TreeWeaver

**Interactive minimum spanning trees from any distance matrix — in your browser.**

TreeWeaver is a single-file, dependency-free web app that turns a pairwise distance
matrix (e.g. SNP distances between genomes) into an interactive minimum spanning tree.
Upload a matrix, watch the tree relax into place, tune the layout, and export a
publication-ready figure — all locally, with no server and no data leaving your machine.

![TreeWeaver example MST](docs/MST_example.png)

*Example: 47 isolates — a 12-sample outbreak core with many offshoots
(see [`examples/example_matrix.tsv`](examples/example_matrix.tsv)).*

---

## Features

- **Upload any distance matrix** (TSV) and build the MST on the fly.
- **Three MST algorithms** — Prim, Kruskal, Boruvka — switchable live.
- **Cluster collapsing** — merge samples within a SNP threshold (0, 1, 2, …) into
  single nodes sized by the number of samples they contain.
- **Four layout modes**:
  - *Spring* — uniform repulsion + SNP-scaled spring rest length.
  - *SNP-weighted repulsion* — repulsion scaled by the SNP distance between nodes.
  - *Stress (MDS)* — Kamada-Kawai stress layout (edge length tracks SNP distance).
  - *Stiff springs* — stronger springs so rest lengths dominate.
- **Live force simulation** with Play/Pause, plus Repulsion / Attraction / Speed sliders.
- **Styling controls** — node-label size, edge-label font size, edge thickness.
- **Export** to **SVG**, **PNG**, and **PDF** (vector), matching exactly what you see.
- **Per-node downloads** — hover a node to download its sample list and its sub-SNP matrix.
- **Fully client-side** — one HTML file, no dependencies, no tracking.

---

## Quick start

### Option 1 — open the file

Download [`treeweaver.html`](treeweaver.html) and open it in any modern browser. That's it.

### Option 2 — GitHub Pages

**GitHub Pages** is GitHub's free static-site hosting: it serves the files of a
repository over the web, so you can use the app without downloading anything.

Once Pages is enabled for this repository (Settings → Pages → *Deploy from a branch* →
`main` / root), the app is available at:

```
https://12nucleus.github.io/TreeWeaver/treeweaver.html
```

> Because the app file is named `treeweaver.html` (not `index.html`), the URL includes
> the file name. If you prefer a clean root URL (`https://12nucleus.github.io/TreeWeaver/`),
> add a tiny `index.html` that redirects to `treeweaver.html`.

### Option 3 — run locally

```bash
git clone https://github.com/12nucleus/TreeWeaver.git
cd TreeWeaver
open treeweaver.html          # macOS
# or: xdg-open treeweaver.html   (Linux)  /  start treeweaver.html  (Windows)
```

Then click **Matrix**, choose a `.tsv` distance matrix, and the tree is built and
animated automatically.

---

## Input format

A square, symmetric distance matrix in **tab-separated** format:

- the first row is a header: an empty corner cell followed by the sample names;
- the first column repeats the sample names;
- the remaining cells are the pairwise distances (integers or floats);
- the diagonal is zero.

```
	ISO-001	ISO-002	ISO-003
ISO-001	0	8	5
ISO-002	8	0	6
ISO-003	5	6	0
```

A ready-to-use example is provided in
[`examples/example_matrix.tsv`](examples/example_matrix.tsv) — 47 isolates forming a
12-sample outbreak core with many offshoots.

---

## Controls

| Control | What it does |
|---|---|
| **Matrix** | Upload a `.tsv` distance matrix. |
| **▶ Play / ⏸ Pause** | Start/stop the live force simulation. |
| **Reset** | Restore the initial layout. |
| **Export + Download** | Save the current view as SVG, PNG, or PDF. |
| **Zoom + / Zoom − / Fit** | Zoom and re-frame the view. |
| **MST** | Choose Prim, Kruskal, or Boruvka. |
| **Collapse** | Merge samples within 0–10 SNPs into single nodes. |
| **Layout** | Spring, SNP-weighted repulsion, Stress (MDS), or Stiff springs. |
| **Repulsion / Attraction / Speed** | Tune the force simulation. |
| **Label size / Edge font / Edge width** | Style the figure. |

**Mouse:** drag a node to pin and move it · drag the background to pan · scroll to zoom ·
hover a node for details and per-node downloads.

---

## Per-node downloads

Hovering a node shows a tooltip with two buttons:

- **Download samples** — a `.txt` with the sample names in that node (one per line).
- **Download matrix** — a `.tsv` with the sub-SNP matrix for that node's members.

Files are named after the uploaded matrix and the node label, e.g.
`example_matrix_ISO-001_6_samples.txt`.

---

## How it works

1. **MST** — the distance matrix is treated as a complete graph; the minimum spanning
   tree is computed with the selected algorithm (see the [addendum](#addendum-mst-algorithms-in-detail)).
2. **Collapse** — samples within the chosen SNP threshold are merged (union-find) into
   super-nodes; edges inside a group are dropped and the minimum edge between groups is kept.
3. **Layout** — a force-directed (or stress) layout positions the nodes; node size grows
   with the square root of the number of samples it represents.
4. **Render** — nodes are drawn as coloured circles, edges are labelled with the SNP
   distance, and everything is interactive.

All of this runs in the browser; the full matrix is embedded in the page (or read from
your uploaded file).

---

## Generating prebuilt pages (optional)

The repository also ships a small Python generator, [`make_mst.py`](make_mst.py), which
can produce a prebuilt page with a matrix already embedded, or the standalone app:

```bash
# standalone app (no embedded matrix)
python3 make_mst.py --app --out-prefix treeweaver

# prebuilt page from a matrix
python3 make_mst.py --matrix examples/example_matrix.tsv --out-prefix example --collapse 0
```

It only needs `numpy` and the standard library.

---

## Addendum: MST algorithms in detail

TreeWeaver builds the tree on the **complete graph** implied by the distance matrix:
every sample is a node, and every pair of samples is an edge weighted by their distance.
A *minimum spanning tree* (MST) is a subset of `n − 1` edges that connects all `n` nodes
with the smallest possible total weight. All three algorithms below produce an MST of the
same total weight; they can differ in *shape* when several edges tie (very common with SNP
data, where many distances are 0 or 1).

### Prim's algorithm (default)

Grows a single tree outward from a starting node.

1. Start with one node in the tree; set `best[v] = distance(start, v)` for every other node.
2. Repeatedly pick the node `u` **not yet in the tree** with the smallest `best[u]`, and add
   the edge that produced it.
3. Relax: for every neighbour `v` of `u`, if `distance(u, v) < best[v]`, update `best[v]` and
   remember `u` as its parent.
4. Stop when all nodes are in the tree.

- **Complexity:** `O(n²)` time with a dense matrix, `O(n)` extra space. This is the fastest
  choice for the dense complete graphs TreeWeaver works with, which is why it is the default.
- **Tie-breaking:** the first minimum found in scan order wins, so equal distances are
  resolved deterministically by node index.

### Kruskal's algorithm

Builds the tree by scanning edges from cheapest to most expensive.

1. Sort **all** edges by weight ascending.
2. Walk the sorted list; add an edge only if its two endpoints are in **different**
   components (checked with a union-find / disjoint-set structure), otherwise skip it
   (it would create a cycle).
3. Stop once `n − 1` edges have been added.

- **Complexity:** `O(E log E)` for the sort plus `O(E α(n))` for union-find, where `E` is the
  number of edges. For a complete graph `E = n(n−1)/2`, so this is `O(n² log n)`.
- **When it shines:** sparse graphs, or when edges are already sorted. It is also the easiest
  to reason about and to parallelise the sort.

### Boruvka's algorithm

Merges components by repeatedly taking each component's cheapest outgoing edge.

1. Start with every node as its own component.
2. In each round, for **every** component find its cheapest edge to a *different* component,
   then add all of those edges at once (merging the components they connect).
3. Repeat until a single component remains.

- **Complexity:** `O(E log n)` = `O(n² log n)` for a complete graph; the number of rounds is
  `O(log n)` because each round at least halves the number of components.
- **When it shines:** it is naturally parallel/distributed (each component can search
  independently), and it is the basis of several fast parallel MST algorithms.

### Which should I use?

| Algorithm | Time (dense) | Best for |
|---|---|---|
| **Prim** | `O(n²)` | dense matrices — the default here |
| **Kruskal** | `O(n² log n)` | sparse graphs, sorted edges, simplicity |
| **Boruvka** | `O(n² log n)` | parallel / distributed settings |

In practice, for a SNP distance matrix all three give the same total weight; switch between
them to explore how ties are broken and how the tree shape changes.

---

## License

[MIT](LICENSE) © 12nucleus
