# Spatial Intelligence — British Museum Floor Plan Analysis

**Marina Osmolovska**
MaCAD 2025 · Semester 3 · Graph ML

---

[View British Museum Ground Floor Map](2D%20files/British_Museum_map.pdf)

---

## About This Analysis

This notebook applies graph-based spatial intelligence methods to the ground floor of the British Museum. The central question is one of accessibility: which rooms and galleries are naturally easy to reach, which attract the most through-traffic simply by being on the way, and which are visually open versus hidden. A secondary thread runs through the shortest path studies — examining routes to destinations in the far corners and periphery of the building, places that visitors often miss not because they aren't looking, but because the spatial configuration works against them.

The analysis uses TopologicPy to build a graph from the floor plan geometry. The floor plan is imported as a mesh, reconstructed into a clean planar face, subdivided with a regular grid, and sliced into a shell of discrete spatial cells. Each cell becomes a node in the graph, and edges connect neighbouring cells. Two graphs are produced: a navigation graph for pathfinding, and an analysis graph for centrality metrics. From these, the analysis derives integration, choice, community structure, degree centrality, and visibility — each revealing a different dimension of how the building works spatially.

---

## The Floor Plan

![Gallery Floor Plan](2D%20files/01_gallery_floor_plan.png)

The starting point is the raw OBJ mesh of the British Museum ground floor, which is cleaned and reconstructed into a single topological face. The result captures the full navigable area of the ground level — including internal voids and courtyards — as a continuous planar shape with collinear edges removed.

---

## Grid Overlay

![Grid Overlay](2D%20files/02_grid_overlay.png)

A regular 3×3 unit grid is clipped to the gallery boundary and overlaid on the floor plan. This grid becomes the basis for spatial subdivision: every cell will represent a discrete location in the graph. The resolution balances detail against computation time — the isovist analysis later in the notebook took approximately 40 minutes at this density.

---

## Sliced Shell

![Sliced Shell](2D%20files/03_sliced_shell.png)

The floor plan is sliced by the grid to produce a shell of individual cells, each assigned a unique identifier. This is the spatial substrate from which all graph-based reasoning follows. What was a continuous floor becomes a network of discrete, connected places.

---

## Analysis Graph

![Analysis Graph](2D%20files/04_analysis_graph.png)

The analysis graph is derived from the shell: each cell becomes a vertex (shown in red) and edges connect adjacent cells. This graph is the core analytical object — it abstracts the physical floor plan into a network structure that supports all the centrality and pathfinding analyses that follow.

---

## Shortest Path Studies

All four path studies compare the raw shortest path through the grid (shown in red) with a straightened version (shown in blue) that eliminates unnecessary turns while staying within the gallery boundary. The straightened path is always shorter and closer to how a person would actually walk. The four destinations were chosen deliberately — they sit in the corners and periphery of the building, representing rooms that are spatially the most difficult to reach from the main entrance.

---

### Montague Place Entrance to Museum Cafe

![Shortest Path: Montague Place to Museum Cafe](2D%20files/05_shortest_path_montague_cafe.png)

Starting from the Montague Place entrance on the upper right and ending at the cafe in the lower left corner, this is the longest route in the study — 298 units along the raw path, reduced to 262 after straightening. The route crosses the full diagonal of the building, which already hints at how much spatial depth separates the two entrances from each other. The 12% reduction from straightening reflects how much the grid introduces unnecessary detours on a diagonal journey.

---

### Main Entrance to Mausoleum of Halikarnassos Room

![Shortest Path: Main Entrance to Mausoleum](2D%20files/06_shortest_path_main_mausoleum.png)

From the Great Russell Street main entrance at the bottom to Room 21 in the upper-left zone — the Mausoleum of Halikarnassos. This room requires navigation through multiple intermediate spaces and sits in a part of the building that visitors rarely pass through by accident. The path is 180 units raw and 157 straightened. Despite being a major collection, the room's position means it is largely dependent on visitors seeking it out with intent.

---

### Main Entrance to Parthenon Sculptures

![Shortest Path: Main Entrance to Parthenon Sculptures](2D%20files/07_shortest_path_main_parthenon.png)

The Parthenon Sculptures are among the most prominent collections in the museum, yet the spatial path to reach them from the main entrance is the longest of the upper-left destinations — 202 units raw, 174 straightened. The 14% gain from straightening is the highest in the study, meaning the grid-based route is particularly indirect here. The room sits slightly deeper in the upper-left zone than the Mausoleum, which explains the added distance despite their proximity on the plan.

---

### Main Entrance to Mexico Exhibition

![Shortest Path: Main Entrance to Mexico Exhibition](2D%20files/08_shortest_path_main_mexico.png)

The Mexico Exhibition is the most accessible of the four corner destinations, with a straightened path of just 138 units from the main entrance. Although it shares the general upper-left zone with the Mausoleum and Parthenon rooms, its position closer to the central axis of the building makes it significantly easier to reach. It is an example of how small differences in location within the same quadrant can translate into meaningful differences in accessibility.

---

### Shortest Path Overview

![Shortest Path Overview](2D%20files/08_shortest_path.png)

A general view of the shortest path routing on the floor plan, showing the navigation graph in use before destination-specific routes are applied.

---

## Closeness Centrality — Integration

![Closeness Centrality](2D%20files/09_closeness_centrality.png)

Closeness centrality measures how close each space is to all other spaces — the fewer steps it takes to reach a cell from anywhere in the building, the higher its score. On the thermal colour scale, warm tones (yellow, orange) indicate high integration and cool tones (purple, dark blue) indicate peripheral, spatially deep locations.

The central corridors and transitional galleries of the museum score highest, confirming their role as the natural hubs of visitor movement. The far corners — including the rooms from the path analysis — score low, which means visitors who happen to be walking through the museum are unlikely to drift toward them naturally. Reaching these rooms requires actively navigating away from the integrated core.

---

## Betweenness Centrality — Choice

![Betweenness Centrality](2D%20files/10_betwenness_centrality.png)

Where closeness centrality describes how easy a place is to reach, betweenness centrality identifies the spaces that most routes pass through — the corridors and junction points that visitors inevitably encounter on the way from anywhere to anywhere else. High-betweenness spaces are the chokepoints of the museum; low-betweenness spaces are bypassed by most routes.

Spaces with low betweenness and low closeness are the most hidden in the building. They are neither natural waypoints nor easy destinations. The peripheral rooms from the path studies fall into this category, meaning they receive neither incidental traffic nor deliberate visits unless a visitor makes a specific effort to find them.

---

## Community Detection

![Community Partition](2D%20files/11_community_partition.png)

Community detection partitions the graph into clusters of cells that are more tightly connected to each other than to the rest of the network. Each colour represents a distinct spatial community. The result reveals the natural zones of the museum — which rooms form coherent districts and how many of these districts the building contains.

The communities correspond roughly to the architectural wings and gallery sequences of the British Museum, validating the graph model against the building's actual spatial logic. Rooms that feel intuitively connected when you walk through the museum tend to belong to the same community in the graph.

![Community Zones](2D%20files/11_community_zones.png)

The individual community zones are extracted as separate face groups, each defined by the outer boundary of its cell cluster. This step converts the cell-level partition into distinct spatial regions with clean perimeters — the building broken into its natural districts.

![Community Graph](2D%20files/11_community_graph.png)

A new graph is derived from the community zones: each zone becomes a single node, and edges connect neighbouring zones. This reduced graph makes the inter-district connectivity legible and is the basis for the degree centrality analysis that follows.

---

## Degree Centrality

![Degree Centrality](2D%20files/12_degree_centrality.png)

Degree centrality is computed at the community level — each detected community is condensed to a single node and its connections to other communities are counted. Communities with high degree are spatial connectors, linking multiple wings and allowing movement in many directions. Communities with low degree are cul-de-sac zones, accessible from fewer sides and typically housing the more peripheral collections.

The rooms from the shortest path studies belong to low-degree communities. Combined with their low closeness and betweenness scores, this paints a consistent picture: the far corners of the museum are not just physically distant from the entrance but structurally isolated in the spatial network.

---

## Visibility — Isovist Analysis

![Isovists](2D%20files/13_isovists.png)

An isovist is the polygon of space visible from a given point. For every vertex in the grid, the isovist is computed — the result is a dense overlay of visibility polygons across the entire floor plan. Some cells near complex boundary conditions returned null isovists and appear empty in the output; these are points where the geometry was too intricate to resolve a valid visible polygon.

![Visibility Heatmap](2D%20files/14_visibility_heatmap.png)

Each isovist is scored by how many other graph vertices fall within it. These scores are interpolated back to the analysis cells and mapped with the thermal colour scale — warm tones (yellow, orange) for high visibility, cool tones (purple, dark blue) for visual isolation.

The most visible spaces are the open, wide areas of the museum — the Great Court, broad corridors, and transitional galleries. The least visible are in alcoves, behind walls, and at the ends of gallery sequences. The peripheral rooms from the path analysis also tend to score low on visibility, adding a further layer to their accessibility disadvantage: they are not only far from the entrance and bypassed by most routes, but also visually cut off from the main circulation areas. A visitor who has never been to the museum would have no way of knowing they are there.

---

## Summary

The analysis consistently identifies a spatial hierarchy in the British Museum's ground floor. The central and transitional spaces — those close to the main entrance, lying on major circulation routes, and opening onto wide areas — score highly across every metric: they are integrated, they see heavy through-traffic, they connect multiple zones, and they are visually open. The peripheral rooms in the far corners score low across all the same metrics. They are spatially deep, bypassed by most routes, structurally isolated in the community graph, and visually hidden.

This matters for collections housed in those peripheral spaces. The Mausoleum of Halikarnassos, the Parthenon Sculptures, and similar rooms in the upper-left wing are major collections, but their location works against visitor discovery. The Mexico Exhibition, sitting slightly closer to the central axis, fares better — an example of how small positional differences within the same building quadrant produce measurable differences in spatial accessibility.

The straightened path lengths from the main entrance give a tangible summary of spatial depth: Mexico Exhibition at 138 units, Mausoleum at 157, Parthenon at 174, and the Museum Cafe via Montague Place at 262. These numbers reflect the lived experience of navigating the museum — the sense that some rooms feel just out of reach no matter which way you turn.
