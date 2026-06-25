# Build Guide — Access-Graph Pipeline (Kyiv block-section)

## Project summary — goals, objectives, narrative

**Course context.** Final project for *Graph Machine Learning (S3 — Buildings as Graphs)*, IAAC MACAD. The brief: recreate a multi-apartment floor plan, convert it to a graph in TopologicPy, run spatial/graph analysis, and run node classification with the pretrained Modified Swiss Dwellings (MSD) model from Seminar 6.

**The story.** A Soviet-era typical panel building (87-02 series row block-section, Kyiv) lost **half of one repeating block-section to a Russian rocket strike**. The staircase core survived, but it now opens onto half-destroyed apartments and a vertical gap where the mirrored apartments used to be. The surviving stair leads, in graph terms, to nothing.

**The proposal — a memorial, not a reconstruction.** The intervention does **not** rebuild the lost flats literally. It preserves their *relationships* as a ghost-like, sculptural public space:
- each lost room returns as a **floor slab only** — no walls; the vanished partitions read as **gaps** (slab perimeters are shrunk by the missing wall thickness);
- the slabs are **redistributed vertically by each room's depth-from-entrance**, so the further a room sat from the apartment door, the lower (or higher) its slab — the walkable promenade literally *becomes the access graph* of the lost apartment;
- the **original door positions are kept as thin bridges** between slabs; because the slabs sit at different heights, those bridges become the **steps/ramps**. Gaps = the walls that were lost; bridges = the doors that are kept.

**What the project actually sets out to do (priorities).**
1. **Primary deliverable — graph construction + spatial analysis.** Build a 3D access graph of the intact block-section in TopologicPy and read its structure (centralities, depth-from-entrance, communities). This analysis *drives the design*: the metrics decide the slab heights.
2. **Design as graph-translation.** Turn the measured relationships of the lost apartment into the memorial's form.
3. **Comparison.** Compare the graph of an intact section (party-wall to party-wall) against the damaged + intervention one, to make the loss legible as topology — the surviving landing as a gateway to an absent community.
4. **Supporting step — node classification (the "box-tick").** Run the pretrained MSD classifier on the **exposed, damaged rooms** to ask what function they now read as, once they open onto the memorial intervention. This is required by the brief but is *not* the centre of gravity — the graph work is.

**Method in one line.** Model one block-section (9 storeys, party-wall to party-wall) as closed cells → access graph via door apertures → centralities + depth-from-entrance → map those values to slab heights (shrunk perimeters, door-bridges as steps) → model the damaged section + intervention → compare intact vs damaged graphs → run the pretrained classifier on the exposed rooms.

---

**Locked decisions:** access graph (doors = edges) · one block-section, party-wall to party-wall · 9 storeys · door positions reused as the steps between shrunken slabs.

**You are reusing your Assignment-1 graph code.** Only two things are new: door *apertures* (to get access instead of adjacency) and a 9-floor *vertical stack* (to get a real 3D graph). Keep that in mind the whole way through.

---

## Which notebooks / files you touch

| File | Role | What you do to it |
|---|---|---|
| **Your A1/A2 TopologicPy notebook** | builds CellComplex → dual graph, runs centralities | adapt it: apertures + vertical stack + access graph |
| **S06-15B GML Prepare Graph for Node Classification.ipynb** | turns a graph into the model's input (features, edge_index, masks) | swap its MSD-sample loader for *your* graph |
| **S06-15C GML Predict MSD Graph.ipynb** | loads `msd_node_classifier.pt`, runs inference, visualises | point it at your prepared graph |
| **Node_Classification_Instructions.docx** | tells you the exact feature schema + class list | **read first** — see "Verify in the notebooks" at the end |
| **msd_node_classifier.pt** | pretrained model | **inference only — never train.** The assignment dropped training. |

The graph **analysis** (centrality, shortest path) lives in *your* notebook, not in S06-15B/C. Those two are only for the ML box-tick at the end.

---

## The exact graph schema (single source of truth)

S06-15B expects two keys to exist *before* its helper runs. Build your TopologicPy graph so every edge and every vertex already carries them, using these **exact lower-case strings**.

### Every edge dict → `door_type`
| your aperture | `door_type` string |
|---|---|
| apartment entrance (landing → hall) | `"entrance_door"` |
| normal room door (leaf door) | `"door"` |
| open threshold / no door leaf (e.g. hall ↔ living opening) | `"passage"` |

The helper turns this into `connectivity` (0/1/2) and the one-hot `feat_connectivity_*`. You don't compute those — you only supply `door_type`.

### Every vertex dict → `room_type`
One of these nine: `bedroom` · `livingroom` · `kitchen` · `dining` · `corridor` · `stairs` · `storeroom` · `bathroom` · `balcony`

**The helper is forgiving about format** (`_clean_string` lower-cases, trims, turns `-`/spaces into `_`) and accepts aliases: `living_room`/`living room`, `stair`/`staircase`, `store_room`/`storage`. So you don't have to be pixel-perfect on those.

**But these have NO mapping and will fail** (label = -1) — convert them yourself:
- your WC / toilet cells → `bathroom`
- your hall / landing / lobby / circulation cells → `corridor`
- your lift → `stairs` (or exclude it from the predicted set)

**Validate before anything else.** Run `CheckMSDGraphPreparation(graph)` — it lists every missing or unsupported `room_type`/`door_type`. Get it to report zero unsupported values before you call `EncodeMSDGraphFeatures`.

### Zoning map (for your reference — the helper does this for you)
| zoning | index | room types |
|---|---:|---|
| private/static | 0 | `bedroom` |
| living/dynamic | 1 | `livingroom`, `kitchen`, `dining`, `corridor` |
| service/functional | 2 | `stairs`, `storeroom`, `bathroom` |
| outdoor/semi-outdoor | 3 | `balcony` |

---

## What the model actually sees — and your two real levers

The helper writes a **7-number node feature vector** and nothing else geometric:
- **zoning one-hot (4)** — derived from `room_type`
- **connectivity proportions (3)** — the mix of `passage`/`door`/`entrance_door` edges touching that room (e.g. 2 doors + 1 passage → `[0.33, 0.67, 0]`)

**There are no area/perimeter features in the model input.** Those stay useful for *your* analysis (Phase 4), but they do **not** feed the classifier. So the prediction is driven only by zoning + the door-type mix + what GraphSAGE aggregates from neighbours.

**Lever 1 — the door types you assign.** Because the node feature is the *proportion* of incident door types, connecting the balcony to an exposed room via `passage` vs `door` vs `entrance_door` literally changes that room's feature vector → changes its prediction. Choose deliberately (an open balcony threshold = `passage`).

**Lever 2 — zoning is an INPUT, and this is the subtle one.** The task is literally "predict fine `room_type` from coarse `zoning` + structure" (the MSD design). That means the provisional `room_type` you give an exposed room sets its zoning, which constrains the answer:

| provisional `room_type` you give the exposed room | its zoning feature | how free the model is |
|---|---|---|
| `bedroom` | `[1,0,0,0]` | **none** — zoning 0 maps only to bedroom, so it just echoes "bedroom" |
| any living/dynamic type (e.g. `corridor`) | `[0,1,0,0]` | **good** — must choose among livingroom/kitchen/dining/corridor; balcony + connectivity decide (in-distribution) |
| an **unsupported placeholder** (e.g. `"unknown"`) | `[0,0,0,0]`, label = -1 | maximum freedom *but* **out-of-distribution** (model never saw all-zero zoning) → unreliable, may also break the loader |

**So your concept ("what do these rooms *become*?") depends on which zoning you grant them — and there's an out-of-distribution catch:**

- During training, **every node had exactly one zoning bit set.** An all-zero vector (from `"unknown"`) is something the model **never saw** → its prediction there is unreliable, and `PyG.ByCSVPath` may also reject `label = -1`. So zeroing is clean in theory but risky in practice.
- **Project decision (faculty-confirmed):** give **all surviving rooms in the damaged block** `room_type = "corridor"` (zoning `[0,1,0,0]`). This applies to every room in both affected flats, not just the ones at the cut edge — both flats lost a room or an entrance, and the design intent is to let the model assign new functions rather than restore old ones. The model must disambiguate among livingroom / kitchen / dining / corridor for each room, decided by connectivity pattern + balcony/stairs neighbours from the intervention.
- **Do not use pre-strike types** for the damaged block rooms — the design explicitly rejects restoring former functions.

This is the seminar's target-leakage point, reframed as a design decision: *neutral zoning for all surviving damaged rooms + balcony intervention elements → the model reads what each space becomes in the memorial context.* Neutral living/dynamic is the correct and deliberate choice here.

---

## Phase 1 — Model the geometry (Rhino/Grasshopper)

### How the cells must behave (read this twice)
- **One room = one closed solid (watertight cell).** A "cell" in TopologicPy is a closed volume.
- **Adjacent rooms share an identical face.** Model rooms edge-to-edge so neighbouring solids share the *same* coincident surface. Do **not** give walls thickness in this model — the shared face represents the wall. Thickness here = a gap = broken adjacency.
- **No overlaps, no gaps.** Overlapping solids and floating gaps both kill the CellComplex.
- **Snap everything.** Shared vertices must be coincident within tolerance (default ~0.0001 m). Turn on snaps; trace shared walls from one shared curve, not two near-identical ones.
- Rooms are the **air volumes people occupy** (room interiors), butted together.

### Recommended Rhino setup (layers)
- `Rooms_F1` … `Rooms_F9` — one closed solid per room per floor
- `Doors` — one marker per door (see Phase 2)
- `Stair`, `Lift` — the vertical-circulation cells
- `Party` — the two party walls that bound the section (your analysis limits)

### Build steps (Grasshopper is better for the 9-floor stack)
1. In Rhino, trace each room of the **typical floor** as a **closed planar curve**, sharing edges with neighbours (trace to wall centrelines so adjacent rooms share the boundary).
2. In GH: `Boundary Surfaces` → `Extrude` (Z = floor height, e.g. 2.8 m) → `Cap Holes` → closed Brep. Each Brep = one cell.
3. **Stack:** `Linear Array` the whole floor up by floor height × 9. Now you have 9 identical floors of cells.
4. The **stair and lift** are their own cells, one per floor, aligned in a vertical shaft.
5. Bake the solids to the right layers (or keep in GH if you'll use the Topologic GH plugin).

> **Scope:** only model from one party wall to the other. That is your "block-section". The party walls are the outer faces — nothing connects through them.

---

## Phase 2 — Doors and vertical access (this is what makes it an *access* graph)

An **aperture** in TopologicPy = a sub-surface on a face that represents an opening (a door). Edges in the access graph are created **only where two cells share an aperture.**

1. **Horizontal doors:** for every real door, place a small face (or a marker you'll convert to a face) **on the shared wall face** between the two rooms it connects. Apartment-entrance doors go on the wall between the **landing** cell and the apartment **hall** cell.
   - **Tag each aperture now with its `door_type`** (`"entrance_door"`, `"door"`, or `"passage"` — see the schema block). The cleanest way: one Rhino sub-layer per type (`Doors_entrance`, `Doors_door`, `Doors_passage`) so the tag is carried by layer and read straight into the edge dict in Phase 3.
2. **No door = no edge.** Two bedrooms sharing a wall with no door get *no* aperture → correctly unconnected.
3. **Vertical access = stairs only.** Two stacked rooms share a *slab*, but you can't walk through a slab, so put **no aperture there**. Put apertures only in the **stair**: one connecting `Landing_F1`↔`Stair_F1`↔`Landing_F2`, and so on up. These become the *only* vertical edges — which is exactly why the core will dominate 3D betweenness. Tag these stair apertures as `"passage"` (they are open stair runs, no door leaf).
4. Windows: **ignore for the access graph.** (They'd matter only for daylight/isovist, which you're not doing.)

> This is also your design hook: the **door positions you place here are the same positions** that will become the step-bridges between shrunken slabs in Phase 5.

---

## Phase 3 — Build the access dual graph (TopologicPy)

Bridge GH/Rhino → TopologicPy by either (a) the **Topologic plugin for Grasshopper** (Brep → Cell components, export the graph as JSON/BRep), or (b) export the solids (OBJ/BRep) and rebuild in the notebook. For Claude Code in VS Code, (b) in the notebook is cleanest.

Reuse your A1 code, with these calls (confirm exact signatures against your A1 notebook + TopologicPy docs):

1. Each room solid → a `Cell`.
2. Add the door faces as **apertures** to their host faces (`Topology.AddApertures(...)`).
3. Merge everything into one **CellComplex** (`CellComplex.ByCells([...])` or `Topology.SelfMerge`).
4. Build the graph **through apertures only**:
   ```
   Graph.ByTopology(
       cellComplex,
       direct=False,            # do NOT connect every shared face
       viaSharedApertures=True, # connect ONLY cells sharing a door
       useInternalVertex=True,  # node sits inside the cell (clean centroid)
       toExteriorApertures=False
   )
   ```
   - `direct=True` would give you the **adjacency** graph (every shared face). You want `False` + `viaSharedApertures=True` for the **access** graph. That one switch is the whole difference.
5. Sanity-check visually (your A1 used a Plotly/PyVista dump). Count edges; confirm bedrooms are leaves and the landing is the hub.

**Write the schema keys into the dictionaries now** (this is the whole interface to S06-15B):
- **each edge dict →** `door_type` = `"entrance_door"` / `"door"` / `"passage"` (from the aperture tag / Rhino sub-layer in Phase 2).
- **each vertex dict →** `room_type` = one of the nine exact strings (map WC→`bathroom`, hall/landing→`corridor`, lift→`stairs`).

**Also store (for your own analysis, not required by the helper):**
- `area`, `perimeter`, `storey`
- room id + apartment id

> If every edge has a valid `door_type` and every vertex a valid `room_type`, the S06-15B helper computes `connectivity`, `feat_connectivity_*`, `label`, `zoning_type`, and `feat_zoning_type_*` automatically. Nothing else is required from you.

---

## Phase 4 — Graph analysis (no grid, no planes — just the graph)

Every metric runs on the node-edge structure. Geometry is irrelevant here. Either use TopologicPy's built-ins or export adjacency to networkx (more flexible for shortest-path-from-a-source).

| Question | Metric | Call |
|---|---|---|
| which rooms are most connected | **degree centrality** | `Graph.Degree` / `nx.degree_centrality` |
| which rooms are most accessible overall | **closeness centrality** | `Graph.ClosenessCentrality` / `nx.closeness_centrality` |
| gateways / circulation spine | **betweenness centrality** | `Graph.BetweennessCentrality` / `nx.betweenness_centrality` |
| **how far each room is from the entrance** (drives your design) | **shortest-path hop-count from the entrance node** | `nx.shortest_path_length(G, source=entrance)` |

- **Unweighted** (count of doors crossed) is cleaner for "depth". Use weighted (real door-to-door distance) only if you want metric distance.
- For the **design driver**, compute **depth-from-entrance inside the lost apartment**: source = that apartment's entrance door node; result = a hop-count per room. Hall ≈ 1, bedrooms/bath ≈ 2–3.
- Run all four on the **whole 9-floor section** too — that's where 3D closeness/betweenness become meaningful (the stair is the only vertical link).

**Conclusion you carry forward:** a ranked list — for each lost room, its depth-from-entrance (and/or closeness).

---

## Phase 5 — Translate the graph into the slabs (the design)

1. Take the **depth-from-entrance** value per lost room.
2. Map it to height: `Z_room = -k × depth` (further from the door = lower; pick `k`, e.g. 0.4 m per hop). Invert the sign if you'd rather *rise* into intimacy.
3. **Shrink each slab's perimeter** inward by the missing wall thickness. The vanished walls now read as **gaps** (wall-ghosts).
4. **Keep a thin bridge exactly where each door was.** This is non-negotiable:
   - bridges are the **only** physical links → they *are* the access-graph edges,
   - people walk the original circulation route,
   - the height difference between two rooms turns each bridge into a **step/ramp** — your "stairs from door placement".
5. Result reads as: **gaps = lost walls, bridges = kept doors, height = graph depth.** The promenade is the graph.

Do this in GH parametrically: feed the per-room depth values (export from Phase 4 as CSV) → drive slab Z and the bridge steps.

---

## Phase 6 — Model the damaged section + intervention as cells

1. Model the **surviving** rooms of the damaged section as cells (the half-apartments at the cut edge).
2. Model your **balcony/slab intervention** as cells too.
3. **Label the intervention.** Both `balcony` and `stairs` are valid classes for this model, so choose by meaning:
   - the open promenade slabs → `balcony` (zoning 3, outdoor/semi-outdoor)
   - the ascending step-bridges → `stairs` (zoning 2, service/functional)
   - mixing is fine and expressive: slabs `balcony`, bridges `stairs`.
   Because `room_type` becomes the zoning input feature, this label **propagates into the model's prediction** for the exposed rooms next to it — that's your intended lever.
4. **Give ALL surviving rooms in the damaged block `room_type = "corridor"`** — not just the rooms at the cut edge, but every room in both affected flats. Faculty direction: predict all remaining rooms in the damaged block, because both flats lost a room or an entrance and their spatial relationships changed. Assigning `corridor` (neutral living/dynamic, in-distribution) preserves the current state without restoring former functions. The model disambiguates among livingroom/kitchen/dining/corridor for each room based on its connectivity and balcony/stairs neighbours. Do NOT use pre-strike types (would echo old function); do NOT use `"unknown"` (out-of-distribution, breaks loader).
5. Put **door apertures** (tagged `door_type`) where the intervention connects to the surviving rooms — these edges carry the balcony/stairs zoning signal into the predictions.
6. Build the access graph for the combined complex (same Phase 3 method).

---

## Phase 7 — Compare intact vs damaged

Run the **same metrics** (Phase 4) on:
- the **intact** section graph, and
- the **damaged + intervention** graph.

Report the deltas: e.g. the surviving Landing keeps high betweenness but now its routes dead-end into the balcony; closeness collapses for the orphaned rooms. This comparison is the analytical heart of the project.

---

## Phase 8 — ML (box-tick: inference only)

The S06-15C flow is now fully known. It predicts **every node** (`split="all"`) — there is **no masking to set up**; you read the predictions for the rooms you care about.

**What gets predicted and what to read:**

| Node type | room_type in graph | y_true in CSV | What to read |
|---|---|---|---|
| Intact section rooms | real type (bedroom, kitchen…) | real type | Sanity check only — not the point |
| Damaged block surviving rooms | `corridor` (placeholder) | `corridor` | **Read y_pred** — this is the model's verdict on their new function |
| Memorial slabs | `balcony` | `balcony` | Echoes design assignment — not informative |
| Step-bridges | `stairs` | `stairs` | Echoes design assignment — not informative |

**Steps:**
1. **At end of Phase 6 (combined graph notebook):** run `CheckMSDGraphPreparation(graph)` → fix until zero unsupported values → run `EncodeMSDGraphFeatures(graph)` → `Graph.ExportToCSV(graph, path=DATASET_PATH)`. This produces `graphs.csv`, `nodes.csv`, `edges.csv`.
2. **Check `nodes.csv` has `train_mask`/`val_mask`/`test_mask` columns.** If `ExportToCSV` omitted them, add `test_mask = True` for every node (prediction uses `split="all"` so values don't matter, but columns must exist).
3. **S06-15C:** update `DATASET_PATH` and `MODEL_PATH` at the top → `PyG.ByCSVPath(...)` → `LoadModel(msd_node_classifier.pt)` → `Predict(split="all", return_probs=True)`.
4. `export_node_predictions(...)` writes `node_predictions_baseline.csv`. **Find surviving damaged-block rooms by `node_id` and read `y_pred`** — that is the model's answer to what each space becomes in the memorial context, with confidence in `y_pred_prob`.
5. Write 3–4 sentences: which functions the damaged rooms were predicted to take on, whether the balcony intervention pushed them toward circulation/living types, and the confidence.

> `y_true` for the damaged block rooms is the `corridor` placeholder — not real ground truth. Do not read accuracy for those nodes. Only `y_pred` is meaningful.

---

## Nothing left open — two things to glance at on the first run

Both notebooks are now fully mapped. On your first pass just confirm:

1. **`nodes.csv` carries the mask columns** (`train_mask`/`val_mask`/`test_mask`). S06-15C reads them; if `ExportToCSV` omitted them, add `test_mask = True` for all nodes. (Prediction is `split="all"`, so the split values don't affect results — the columns just need to exist.)
2. **You're reading `y_pred`, not accuracy,** for the exposed rooms — their `y_true` is the provisional label you assigned, not a real ground truth.

Everything else — the nine room types, three door types, four zoning classes, the 7-feature node vector (zoning 4 + connectivity 3), no geometric inputs, predict-all inference — is confirmed from the two notebooks.

---

## One-line order of operations
Model intact section (9 floors, doors, stair) → CellComplex → **access** graph → centralities + depth-from-entrance → map depth to slab heights (shrink perimeter, door-bridges = steps) → model combined section (intact rooms with real types + ALL surviving damaged-block rooms as `corridor` + memorial slabs as `balcony` + bridges as `stairs`) → combined access graph → compare intact vs combined metrics → S06-15B (CheckMSD + EncodeMSD + ExportToCSV) → S06-15C predict → read y_pred for surviving damaged-block rooms.
