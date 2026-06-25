# Project Handoff — Marina Osmolovska, MaCAD S3 Graph ML Final

Paste this entire document into a new Claude Code session to resume work.

---

## Who I am and what this project is

Marina Osmolovska, IAAC MaCAD 2025, Semester 3 Graph ML final project.

**Building:** Soviet-era 87-02 panel block-section (Р-2.2.3.3 рядовая), Kyiv, 9 storeys. Half destroyed by a Russian strike. The exposed staircase survived and leads to: one fully intact apartment, one partial apartment (missing one room).

**Design concept:** Memorial intervention — rooms that were lost return as depth-mapped floor slabs (slab Z lowers the further from the entrance). Door positions become step-bridges. Graph analysis drives the spatial design.

**Goal:** Build an access graph of the intact section in TopologicPy → analyse it with NetworkX → design the intervention → predict room types in damaged/intervention zones using pretrained ML model (`msd_node_classifier.pt`).

---

## Where I am in the pipeline

**Phases 1–2 are DONE.** I have modeled rooms as closed solid boxes in Rhino (shared faces = shared walls, zero gap) and placed door aperture faces on shared walls. I have exported 4 OBJ files.

**Next task: Phase 3 — build the TopologicPy notebook** `Final/MO_Final_Phase3_AccessGraph.ipynb`.

**ML scope (faculty-confirmed):** ML inference runs **only on the combined graph** (Phase 6 notebook), not on the intact section. The intact section is fully known — all rooms have their real types, nothing to predict. Within the combined graph, meaningful predictions are for **all surviving rooms in both affected flats of the damaged block** (not just edge rooms). Those rooms are assigned `room_type="corridor"` as a neutral placeholder — the model predicts their new function given changed topology and new connections to the memorial intervention. Intervention slabs = `balcony`, bridges = `stairs` — design-assigned, not predicted.

---

## The 4 OBJ files I exported

All are in the same folder as the notebooks (`Final/` or nearby — confirm exact path at start of session).

| File | Contents | Export settings used |
|---|---|---|
| `rooms.obj` | All room boxes, each named by room_type (Name property in Rhino Object Properties) | "As OBJ objects" |
| `doors_entrance.obj` | Entrance door aperture faces only (apartment entrance from landing) | Any naming (type assigned in code) |
| `doors_door.obj` | Internal door aperture faces (room-to-room) | Any naming |
| `doors_passage.obj` | Passage aperture faces (stair landing to corridor, no physical door) | Any naming |

**Room names used in Rhino (exact strings, must match):**
`bedroom`, `livingroom`, `kitchen`, `dining`, `corridor`, `stairs`, `storeroom`, `bathroom`, `balcony`

The staircase is **ONE TALL BOX** spanning the full building height (not per-floor boxes). Vertical access within the staircase box = passage apertures on the vertical walls between staircase and each floor's landing (2 per floor × 9 floors = 18 apertures, all in `doors_passage.obj`).

---

## Key repo files

Working directory: `d:\Marina\MaCAD 2025\3 SEMESTER\Graph ML\graph_ml\graph_ml\`

| File | Role |
|---|---|
| `Final/topology.ipynb` | Working executed code — source of all confirmed API patterns |
| `Final/pipeline_build_guide.md` | Full 8-phase pipeline spec (reference doc) |
| `Final/S06-15B GML Prepare Graph for Node Classification.ipynb` | Contains `EncodeMSDGraphFeatures()` and `CheckMSDGraphPreparation()` |
| `Final/S06-15C GML Predict MSD Graph.ipynb` | Inference: loads CSVs → model → predictions |
| `Final/msd_node_classifier.pt` | Pretrained GraphSAGE model (do not retrain) |
| `A1_Topology/GraphML_MarinaOsmolovska_A1.ipynb` | A1 reference — all rooms in one OBJ, doors in separate OBJ |

---

## Confirmed TopologicPy API patterns (from topology.ipynb — working, executed)

```python
# Load OBJ — no transposeAxes argument needed
objects = Topology.ByOBJPath("path/to/rooms.obj")

# Get name from dictionary
d = Topology.Dictionary(obj)
name = Dictionary.ValueAtKey(d, "name")   # reads the 'o' line from OBJ
group = Dictionary.ValueAtKey(d, "group") # reads the 'g' line from OBJ

# Build Cell from flat-face object
faces = Topology.Faces(obj)
if len(faces) > 1:
    cell = Cell.ByFaces(faces)
else:
    cell = faces[0]
cell = Topology.RemoveCollinearEdges(cell)

# Set dictionary on a vertex selector
selector = Topology.InternalVertex(cell)
d2 = Dictionary.SetValuesAtKeys(d, ["room_type", "label"], ["bedroom", 0])
selector = Topology.SetDictionary(selector, d2)

# Build CellComplex
house = CellComplex.ByCells(cells)

# Transfer dictionaries from selectors to cells
house = Topology.TransferDictionariesBySelectors(house, selectors, tranCells=True)

# Add apertures (door faces)
house = Topology.AddApertures(house, apertures, subTopologyType="face")

# Build ACCESS graph (not adjacency — edges only through door apertures)
g = Graph.ByTopology(
    house,
    direct=False,
    viaSharedApertures=True,
    toExteriorApertures=False,
    useInternalVertex=True,
    storeBRep=False,
    tolerance=0.0001
)
```

---

## Room type → label → zoning schema (single source of truth)

| room_type | label | zoning_class | zoning meaning |
|---|---|---|---|
| bedroom | 0 | 0 | private/static |
| livingroom | 1 | 1 | living/dynamic |
| kitchen | 2 | 1 | living/dynamic |
| dining | 3 | 1 | living/dynamic |
| corridor | 4 | 1 | living/dynamic |
| stairs | 5 | 2 | service/functional |
| storeroom | 6 | 2 | service/functional |
| bathroom | 7 | 2 | service/functional |
| balcony | 8 | 3 | outdoor |

WC/toilet → `bathroom`. Hall/landing/lobby → `corridor`. Lift shaft → `stairs`.

---

## Door type → connectivity schema

| door_type | connectivity_value |
|---|---|
| passage | 0 |
| door | 1 |
| entrance_door | 2 |

Stair landing to corridor: `passage`. Apartment entrance from landing: `entrance_door`. Room-to-room: `door`. Room to balcony: `door`.

---

## 7-feature node vector (what the pretrained model sees)

- `feat_zoning_type_0` through `feat_zoning_type_3` — one-hot encoding of zoning_class (4 dims)
- `feat_connectivity_0` — proportion of incident edges that are `passage` (0)
- `feat_connectivity_1` — proportion of incident edges that are `door` (1)
- `feat_connectivity_2` — proportion of incident edges that are `entrance_door` (2)

No geometry, no area, no perimeter in model input.

---

## CSV schema required by S06-15C

**graphs.csv:** `graph_id, num_nodes`

**nodes.csv:** `graph_id, node_id, label, feat_zoning_type_0, feat_zoning_type_1, feat_zoning_type_2, feat_zoning_type_3, feat_connectivity_0, feat_connectivity_1, feat_connectivity_2, train_mask, val_mask, test_mask`

**edges.csv:** `graph_id, src_id, dst_id, feat_connectivity_0, feat_connectivity_1, feat_connectivity_2`

If `Graph.ExportToCSV` omits mask columns, add `train_mask=False, val_mask=False, test_mask=True` for all nodes. S06-15C uses `split="all"` so mask values don't affect predictions — columns just must exist.

---

## Design decisions already made

1. **Staircase = one tall box**, full building height. Not per-floor. This is correct for the project narrative and consistent with A1/A3 core pattern.

2. **All surviving rooms in the damaged block** (both affected flats, all floors — not just edge rooms) get `room_type="corridor"` as a neutral placeholder. The design intent is not to restore their former functions: the model predicts what they read as given the changed topology and their new connections to the memorial. Do NOT assign their pre-strike room types, and do NOT use `"unknown"` (out-of-distribution, may break the loader).

3. **Intervention slabs** (memorial elements at depth-mapped Z heights, replacing completely destroyed rooms) → `room_type="balcony"` (zoning 3, outdoor/semi-outdoor — architecturally accurate for open-sky slabs; propagates outdoor-neighbour signal into adjacent surviving rooms via GraphSAGE aggregation). Step-bridge connectors → `room_type="stairs"`.

4. **Balcony door type** → `door` (same as room-to-room internal door).

5. **Phase 5 design formula:** `slab_Z = -SLAB_STEP_M × depth_from_entrance` where depth = NetworkX shortest path length from entrance node.

6. **Staircase apertures:** 18 `passage` apertures total (2 per floor × 9 floors) on the vertical walls of the tall staircase box. Drew 2 at Z=0, used Linear Array with Z step = floor_height, count = 8 additional floors.

7. **4 OBJ files total** (not 11, not per-floor).

---

## The notebook to create now

**File:** `Final/MO_Final_Phase3_AccessGraph.ipynb`

This notebook covers Phases 3 + 4 of the pipeline only:
- Phase 3: Load OBJs → build Cells → build CellComplex → add apertures → build access graph (+ adjacency graph for comparison)
- Phase 4: Convert to NetworkX → depth from entrance → centrality metrics → print table for design use

Phase 8 (encode + export CSV) happens in the **Phase 6 combined-graph notebook**, not here.

**Structure of notebook:**

### Cell 1 — Imports
```python
import topologicpy
from topologicpy.Topology import Topology
from topologicpy.Cell import Cell
from topologicpy.CellComplex import CellComplex
from topologicpy.Graph import Graph
from topologicpy.Dictionary import Dictionary
import networkx as nx
import pandas as pd
import os
```

### Cell 2 — Paths
```python
BASE = os.path.dirname(os.path.abspath("__file__"))  # adjust if needed
ROOMS_OBJ    = os.path.join(BASE, "rooms.obj")
DOORS_ENT    = os.path.join(BASE, "doors_entrance.obj")
DOORS_DOOR   = os.path.join(BASE, "doors_door.obj")
DOORS_PASS   = os.path.join(BASE, "doors_passage.obj")
OUTPUT_DIR   = os.path.join(BASE, "graph_output")
os.makedirs(OUTPUT_DIR, exist_ok=True)
```

### Cell 3 — Room type mapping helpers
```python
ROOM_LABEL = {
    "bedroom":0,"livingroom":1,"kitchen":2,"dining":3,"corridor":4,
    "stairs":5,"storeroom":6,"bathroom":7,"balcony":8
}
ROOM_ZONING = {
    "bedroom":0,"livingroom":1,"kitchen":1,"dining":1,"corridor":1,
    "stairs":2,"storeroom":2,"bathroom":2,"balcony":3
}
DOOR_CONN = {"passage":0,"door":1,"entrance_door":2}
```

### Cell 4 — Load rooms, build cells
```python
raw_rooms = Topology.ByOBJPath(ROOMS_OBJ)
cells, selectors = [], []
for obj in raw_rooms:
    d = Topology.Dictionary(obj)
    room_type = (Dictionary.ValueAtKey(d, "name") or "").strip().lower()
    if room_type not in ROOM_LABEL:
        print(f"WARNING: unknown room_type '{room_type}' — skipping")
        continue
    faces = Topology.Faces(obj)
    cell = Cell.ByFaces(faces) if len(faces) > 1 else faces[0]
    cell = Topology.RemoveCollinearEdges(cell)
    label = ROOM_LABEL[room_type]
    zoning = ROOM_ZONING[room_type]
    d2 = Dictionary.SetValuesAtKeys(d, ["room_type","label","zoning"], [room_type, label, zoning])
    selector = Topology.InternalVertex(cell)
    selector = Topology.SetDictionary(selector, d2)
    cells.append(cell)
    selectors.append(selector)
print(f"Loaded {len(cells)} cells")
```

### Cell 5 — Build CellComplex
```python
house = CellComplex.ByCells(cells)
house = Topology.TransferDictionariesBySelectors(house, selectors, tranCells=True)
print("CellComplex built")
```

### Cell 6 — Load apertures
```python
def load_apertures(path, door_type):
    objs = Topology.ByOBJPath(path)
    result = []
    for obj in objs:
        d = Topology.Dictionary(obj)
        conn = DOOR_CONN[door_type]
        d2 = Dictionary.SetValuesAtKeys(d, ["door_type","connectivity"], [door_type, conn])
        ap = Topology.SetDictionary(obj, d2)
        result.append(ap)
    return result

apertures = (
    load_apertures(DOORS_ENT,  "entrance_door") +
    load_apertures(DOORS_DOOR, "door") +
    load_apertures(DOORS_PASS, "passage")
)
print(f"Loaded {len(apertures)} apertures")
```

### Cell 7 — Add apertures, build access graph
```python
house = Topology.AddApertures(house, apertures, subTopologyType="face")
g = Graph.ByTopology(
    house,
    direct=False,
    viaSharedApertures=True,
    toExteriorApertures=False,
    useInternalVertex=True,
    storeBRep=False,
    tolerance=0.0001
)
print(f"Graph: {Graph.Order(g)} nodes, {Graph.Size(g)} edges")
```

### Cell 8 — Convert to NetworkX, find entrance, compute depth
```python
# Convert to NetworkX
nx_g = Graph.NetworkXGraph(g)  # or use Graph.ExportToNetworkX(g) depending on API

# Find entrance node (the node with an entrance_door edge)
entrance_node = None
for u, v, data in nx_g.edges(data=True):
    if data.get("door_type") == "entrance_door":
        # entrance is on the landing/staircase side
        # check which node is the staircase
        for n in [u, v]:
            rt = nx_g.nodes[n].get("room_type","")
            if rt in ("corridor","stairs"):
                entrance_node = n
                break
    if entrance_node:
        break

print(f"Entrance node: {entrance_node}, room_type: {nx_g.nodes[entrance_node].get('room_type')}")

# Depth from entrance
depth = nx.single_source_shortest_path_length(nx_g, entrance_node)
nx.set_node_attributes(nx_g, depth, "depth_from_entrance")
```

### Cell 9 — Centrality
```python
betweenness = nx.betweenness_centrality(nx_g)
closeness   = nx.closeness_centrality(nx_g)
degree      = nx.degree_centrality(nx_g)
nx.set_node_attributes(nx_g, betweenness,  "betweenness")
nx.set_node_attributes(nx_g, closeness,    "closeness")
nx.set_node_attributes(nx_g, degree,       "degree_centrality")

for n, data in nx_g.nodes(data=True):
    print(n, data.get("room_type"), "depth:", data.get("depth_from_entrance"), 
          f"btw:{betweenness[n]:.3f}")
```

### Cell 10 — Encode MSD features
```python
def encode_msd_features(nx_g):
    rows = []
    for node_id, data in nx_g.nodes(data=True):
        room_type = data.get("room_type", "corridor")
        zoning    = ROOM_ZONING.get(room_type, 1)
        label     = ROOM_LABEL.get(room_type, 4)
        zoning_oh = [1 if zoning==i else 0 for i in range(4)]
        edges = list(nx_g.edges(node_id, data=True))
        total = len(edges) or 1
        conn_counts = [0, 0, 0]
        for _, _, ed in edges:
            c = DOOR_CONN.get(ed.get("door_type","door"), 1)
            conn_counts[c] += 1
        conn_props = [c/total for c in conn_counts]
        rows.append({
            "graph_id": 0,
            "node_id": node_id,
            "label": label,
            "feat_zoning_type_0": zoning_oh[0],
            "feat_zoning_type_1": zoning_oh[1],
            "feat_zoning_type_2": zoning_oh[2],
            "feat_zoning_type_3": zoning_oh[3],
            "feat_connectivity_0": conn_props[0],
            "feat_connectivity_1": conn_props[1],
            "feat_connectivity_2": conn_props[2],
            "train_mask": False,
            "val_mask":   False,
            "test_mask":  True,
            "depth_from_entrance": data.get("depth_from_entrance", -1),
            "room_type": room_type,
        })
    return pd.DataFrame(rows)

nodes_df = encode_msd_features(nx_g)
print(nodes_df[["node_id","room_type","label","feat_zoning_type_0","feat_connectivity_1"]].head(10))
```

### Cell 11 — Export CSVs
```python
# graphs.csv
graphs_df = pd.DataFrame([{"graph_id": 0, "num_nodes": len(nodes_df)}])
graphs_df.to_csv(os.path.join(OUTPUT_DIR, "graphs.csv"), index=False)

# nodes.csv (ML columns only, no extra columns)
ml_cols = ["graph_id","node_id","label",
           "feat_zoning_type_0","feat_zoning_type_1","feat_zoning_type_2","feat_zoning_type_3",
           "feat_connectivity_0","feat_connectivity_1","feat_connectivity_2",
           "train_mask","val_mask","test_mask"]
nodes_df[ml_cols].to_csv(os.path.join(OUTPUT_DIR, "nodes.csv"), index=False)

# edges.csv
edge_rows = []
for u, v, data in nx_g.edges(data=True):
    conn = DOOR_CONN.get(data.get("door_type","door"), 1)
    conn_oh = [1 if conn==i else 0 for i in range(3)]
    edge_rows.append({
        "graph_id": 0, "src_id": u, "dst_id": v,
        "feat_connectivity_0": conn_oh[0],
        "feat_connectivity_1": conn_oh[1],
        "feat_connectivity_2": conn_oh[2],
    })
edges_df = pd.DataFrame(edge_rows)
edges_df.to_csv(os.path.join(OUTPUT_DIR, "edges.csv"), index=False)

print("Exported to:", OUTPUT_DIR)
print(f"  graphs.csv: {len(graphs_df)} rows")
print(f"  nodes.csv:  {len(nodes_df)} rows")
print(f"  edges.csv:  {len(edges_df)} rows")
```

---

## What comes after Phase 3 notebook

**Phase 4 (analysis)** is the second half of the Phase 3 notebook (NetworkX cells).
Output: a table of `room_name | depth_from_entrance | betweenness | closeness`. Write these down — they drive the Rhino design.

**Phase 5 (design — Rhino):** Use the depth table → `slab_Z = -SLAB_STEP_M × depth_from_entrance`. Model:
- Memorial slabs (replacing destroyed rooms) as `balcony` boxes at the computed Z heights, perimeters shrunk by wall thickness
- Step-bridges (at original door positions) as `stairs` boxes
- All surviving rooms in both damaged flats named `corridor` (neutral placeholder)
- Door aperture faces on all new connections

**Phase 6 (combined Rhino export):** Export from Rhino:
- Combined `rooms_combined.obj` — intact section rooms (real types) + damaged block all surviving rooms (`corridor`) + memorial slabs (`balcony`) + bridges (`stairs`)
- Combined door aperture OBJs (reuse intact-section files + new ones for intervention connections)

**Phase 7 (combined graph notebook — `MO_Final_Phase6_CombinedGraph.ipynb`):**
Same structure as Phase 3 notebook, pointing at combined OBJs. At the end, adds three cells:
```python
%run "S06-15B GML Prepare Graph for Node Classification.ipynb"
CheckMSDGraphPreparation(g)
g = EncodeMSDGraphFeatures(g)
Graph.ExportToCSV(g, path=r"...\Final\dataset")
```
Also re-runs the Phase 4 metrics on the combined graph and compares against the intact-section results (the comparison is the analytical heart of the project).

**Phase 8 (ML inference — S06-15C):**
Update two paths at the top, run all cells:
```python
DATASET_PATH = Path(r"D:\Marina\...\Final\dataset")
MODEL_PATH   = Path(r"D:\Marina\...\Final\msd_node_classifier.pt")
```
Output: `node_predictions_baseline.csv` with `y_pred` per node.
- **Read `y_pred` for the surviving damaged-block rooms** — these are the meaningful predictions (their `y_true` is the `corridor` placeholder, not real ground truth).
- Intact section predictions can be checked for sanity but are not the point.
- Intervention slabs/bridges predictions just echo the design-assigned type.

---

## Common issues to watch for

- **CellComplex fails:** rooms must share exactly coincident faces — zero gap, zero overlap. If boxes were modeled with wall thickness, the face-to-face share may not be exact. Test with 2-3 rooms first.
- **Graph has no edges:** means apertures didn't attach to host faces. Check aperture faces are exactly coplanar with room faces (same Z, same X/Y extents, inside the face boundary).
- **Graph.NetworkXGraph vs Graph.ExportToNetworkX:** API name may vary by topologicpy version — check what's available with `dir(Graph)` if one fails.
- **`name` key empty:** means Rhino objects weren't named before OBJ export. Check `Dictionary.ValueAtKey(d, "group")` as fallback — if layer names were exported as groups, group key has the room type.
- **S06-15C loader error on mask columns:** add `test_mask=True` for all nodes in nodes.csv — see Cell 11 above.
