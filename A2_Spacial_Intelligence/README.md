# A2 — Spatial Intelligence: British Museum Floor Plan Analysis

**Author:** Marina Osmolovska
**Course:** Graph ML — MaCAD 2025, Semester 3
**Tool:** TopologicPy + Plotly, Python

---

## British Museum Ground Floor Map

> Reference floor plan used to define the spatial boundary for all analyses.

[View British Museum Floor Plan PDF](2D%20files/British_Museum_map.pdf)

---

## Workflow

The analysis applies graph-based spatial intelligence methods to the British Museum ground floor plan. The floor plan geometry is imported as a 3D mesh, cleaned into a planar face, overlaid with a regular grid, and sliced into a shell of discrete spatial cells. Two graphs are derived from this shell:

- **Navigation graph** — connects cells via shared edges (walls/openings), used for pathfinding
- **Analysis graph** — connects cells via shared topology, used for centrality metrics

The core research question is **accessibility**: which rooms and corridors are easiest to reach, how much through-traffic they attract, and how visible they are from any given point. The four shortest path case studies specifically examine routes to destinations in the far corners of the building — areas that are spatially deepest and most likely to be missed by visitors.

---

## Analysis

### 1. Gallery Floor Plan Import

![Gallery Floor Plan](2D%20files/01_gallery_floor_plan.png)

The raw OBJ mesh of the British Museum ground floor is imported and reconstructed as a clean planar face, with collinear edges removed. The result is a single topological face representing the navigable floor area — including internal courtyards and voids — rendered against a black background to emphasise the building boundary.

---

### 2. Grid Overlay

![Grid Overlay](2D%20files/02_grid_overlay.png)

A regular 3×3 unit grid is clipped to the gallery boundary. The grid serves as the spatial subdivision basis for the analysis: each cell will become a discrete node in the graph. Grid density balances computational cost against spatial resolution — finer grids capture more detail but significantly increase isovist computation time.

---

### 3. Sliced Shell

![Sliced Shell](2D%20files/03_sliced_shell.png)

The floor plan face is sliced by the grid to produce a **topological shell** of individual rectangular cells. Each cell is assigned a unique `face_id`. This shell is the spatial substrate from which both the navigation and analysis graphs are derived.

---

### 4. Analysis Graph

![Analysis Graph](2D%20files/04_analysis_graph.png)

The analysis graph is derived from the shell: each cell becomes a vertex (red dot) and edges connect adjacent cells. This graph abstracts the continuous floor plan into a network structure, enabling all subsequent graph-theoretic metrics. The graph captures how spaces are topologically connected, independent of their physical geometry.

---

## Shortest Path Analysis

All four paths compare the **original shortest path** (red) against a **straightened/optimised path** (blue) that avoids unnecessary turns while remaining within the gallery boundary. The straightened path is shorter in length and more legible as a wayfinding route.

The case studies target rooms in the **far corners and periphery** of the building — destinations that real visitors often struggle to find. The path lengths reveal how spatially deep these rooms are.

---

### 5. Montague Place Entrance → Museum Cafe

![Shortest Path: Montague Place to Museum Cafe](2D%20files/05_shortest_path_montague_cafe.png)

| | Length |
|---|---|
| Original path | 298.61 units |
| Straightened path | 261.58 units |

Starting from the **Montague Place entrance** (upper right), this route crosses the full depth of the museum to reach the **cafe in the lower left corner**. It is the longest path analysed, reflecting the extreme diagonal distance between these two points. The 12% reduction from straightening shows significant redundancy in the grid-based shortest path.

---

### 6. Main Entrance → Mausoleum of Halikarnassos Room

![Shortest Path: Main Entrance to Mausoleum](2D%20files/06_shortest_path_main_mausoleum.png)

| | Length |
|---|---|
| Original path | 180.43 units |
| Straightened path | 156.63 units |

From the **Great Russell Street main entrance** (bottom) to **Room 21** in the upper-left zone — home to the Mausoleum of Halikarnassos sculptures. This room sits in a topologically peripheral position: it requires navigation through multiple intermediate spaces and is unlikely to be encountered by visitors who do not seek it out deliberately.

---

### 7. Main Entrance → Parthenon Sculptures

![Shortest Path: Main Entrance to Parthenon Sculptures](2D%20files/07_shortest_path_main_parthenon.png)

| | Length |
|---|---|
| Original path | 201.79 units |
| Straightened path | 174.46 units |

Route to the **Parthenon Sculptures** (upper-left gallery area). Despite being one of the museum's most prominent collections, the spatial path is longer than the Mausoleum route, suggesting the Parthenon room sits slightly deeper in the upper-left zone. The 14% straightening gain is the largest among the four routes, indicating the grid-based path takes a particularly indirect approach.

---

### 8. Main Entrance → Mexico Exhibition

![Shortest Path: Main Entrance to Mexico Exhibition](2D%20files/08_shortest_path_main_mexico.png)

| | Length |
|---|---|
| Original path | 160.42 units |
| Straightened path | 138.32 units |

The **Mexico Exhibition** is the most accessible of the four corner destinations, with the shortest path from the main entrance (160 units vs. 298 for the café). Its position in the upper-left zone — closer to the central axis — makes it more reachable than the Mausoleum or Parthenon rooms despite sharing the same general quadrant.

---

## Centrality & Spatial Structure Analysis

---

### 9. Closeness Centrality (Integration)

![Closeness Centrality](2D%20files/09_closeness_centrality.png)

**Hot (bright) = high integration / easy to reach from everywhere**
**Cool (dark) = peripheral / spatially deep**

Closeness centrality measures how close each space is to all other spaces — i.e. how few steps it takes to reach it from anywhere in the building. High-integration spaces (warm colours on the thermal scale) are the natural hubs of visitor flow. In a museum context, these are the rooms visitors pass through most often, regardless of intent.

The central corridor and transitional gallery zones typically score highest. The far corners — including the rooms targeted in the shortest path analysis — show low integration, confirming that these destinations are genuinely hard to reach and unlikely to receive incidental visitors.

---

### 10. Betweenness Centrality (Choice)

![Betweenness Centrality](2D%20files/10_betweenness_centrality.png)

**High betweenness = many shortest paths pass through this space**
**Low betweenness = bypassed by most routes**

Betweenness centrality identifies the **chokepoints and corridors** of the museum: spaces that appear on the highest number of shortest routes between any two cells. These are the spaces visitors inevitably pass through, regardless of their destination.

Rooms with low betweenness but reasonable closeness are "pleasant side destinations" — accessible but not obligatory. Rooms with both low closeness and low betweenness (deep corners) are the most hidden spaces in the museum, capturing the rooms from the shortest path analysis.

---

### 11. Community Detection

![Community Detection](2D%20files/11_community_detection.png)

The museum is partitioned into **spatial communities** — groups of cells that are more strongly connected to each other than to the rest of the network. Each colour represents a distinct community. This reveals the natural spatial zones of the building: which rooms form coherent clusters and how many natural "districts" the museum contains.

Communities roughly correspond to the architectural wings and gallery sequences of the British Museum, validating the graph model against the real building logic.

---

### 12. Degree Centrality

![Degree Centrality](2D%20files/12_degree_centrality.png)

Computed on the **community-level graph** (each community condensed to a single node), degree centrality measures how many other communities each zone connects to. High-degree communities are spatial **connectors** — transitional zones linking multiple wings. Low-degree communities are **cul-de-sac wings**, accessible from fewer directions and typically housing the more peripheral collections.

The rooms from the shortest path case studies fall within low-degree communities, consistent with their deep spatial position.

---

### 13. Visibility / Isovist Analysis

![Visibility Analysis](2D%20files/13_visibility_isovist.png)

> Note: Isovist computation took ~40 minutes. Some grid cells near complex boundaries returned null isovists and appear empty in the output.

**Hot = high visibility / can see many other cells from here**
**Cool = low visibility / visually isolated**

An **isovist** is the polygon of space visible from a given point. For each grid vertex, the isovist is computed and the number of other graph vertices it contains is counted as the visibility score. This is interpolated back to the analysis graph cells.

High-visibility spaces are open, central areas — the Great Court, wide corridors, and transitional galleries. Low-visibility spaces are tucked behind walls, in alcoves, or at the ends of gallery sequences. The far-corner rooms from the path analysis also tend to have restricted visibility, compounding their accessibility disadvantage: not only are they far from the entrance, they are also visually hidden from the main circulation areas.

---

## Summary

| Metric | Most Accessible Spaces | Least Accessible Spaces |
|---|---|---|
| Closeness (Integration) | Central corridors, main hall | Far corners (Mausoleum, Parthenon rooms) |
| Betweenness (Choice) | Main circulation spine | Side galleries, peripheral wings |
| Community | Connector zones | Isolated wings |
| Visibility | Open courts, wide halls | Alcoves, corner galleries |
| Shortest path (from main entrance) | Mexico Exhibition (138 units) | Museum Cafe via Montague Pl (262 units) |

The analysis confirms a consistent spatial hierarchy: the **central and transitional spaces** of the British Museum score high across all accessibility metrics, while the **peripheral corner rooms** — despite housing major collections — are spatially deep, visually hidden, and unlikely to be discovered without deliberate navigation.
