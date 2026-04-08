# T-Clip FreeCAD Project — Session Log

## Project Summary

Designed a **T-shaped PCB clip** in FreeCAD using Python/OCC via MCP tools.
The clip holds two PCBs together at 90°: a horizontal base PCB and a vertical PCB standing orthogonally on it.

---

## Final Model: `T_Clip_v4`

**File:** `t-clip.FCStd` (FreeCAD document named "Scene", object "T_Clip_v4")

### Dimensions

| Parameter | Value |
|-----------|-------|
| PCB thickness | 1.5mm |
| Wall thickness | 2.0mm |
| Arm width (arm_w) | 5.5mm (2×wall + pcb_t) |
| Horizontal arm length | 20mm |
| Vertical arm length | 10mm |
| Total depth (Z) | 8mm |
| Stop wall | 2mm solid at Z=6–8mm |
| Slot depth | 6mm (Z=0 to Z=6) |
| Outer fillet radius | 1.2mm |

### Coordinate System

- Y=0: bottom of horizontal (base PCB) slot
- Y=1.5: top of base PCB = bottom of vertical PCB (zero-gap corner)
- Y=3.5: junction between H-arm and V-arm (junc_y = pcb_t + wall)
- Z=0: open insertion side
- Z=6–8: solid stop wall
- arm_hx = arm_w/2 = 2.75mm (half-width of arm in X)

### Slots

- **H-slot**: Y=0 to Y=1.5, Z=0 to Z=6, open at both X ends → base PCB slides in from either side
- **V-slot**: X=−0.75 to X=+0.75, Y=1.5 to Y=open, Z=0 to Z=6 → vertical PCB slides down

### Internal Grip Ridges (11 total, 0.3mm high × 0.5mm wide × 6mm long)

| Location | Positions |
|----------|-----------|
| H-slot bottom (Y=0, ridge protrudes +Y) | X = −6, 0, +6 |
| H-slot top (Y=1.5, ridge protrudes −Y) | X = −6, +6 (skip X=0 — vert slot zone) |
| V-slot left wall (X=−0.75, protrudes +X) | Y = 5.5, 8.5, 11.5 |
| V-slot right wall (X=+0.75, protrudes −X) | Y = 5.5, 8.5, 11.5 |

### Key Design Constraints

- **Zero-gap corner**: V-slot starts at Y=pcb_t=1.5 (top of base PCB) — no bridging material; both PCBs remain in direct contact
- **C-channel style**: H-slot open at both X ends, V-slot open at top, both open at Z=0
- **Stop wall**: slot cuts only to Z=6, leaving solid 2mm back wall

---

## Build Process (Final Working Code)

### Critical Fix: Sharp Inside Junction Corners

**Problem (v3 and earlier):** Filleting both arm boxes on ALL edges before fusing created concave "carved-in" appearance at the inside corners of the T-junction. User specifically flagged this as wrong.

**Root cause:** When H-arm top edges (at Y=3.5) in the junction zone were filleted, the rounded corners became concave after fusing with V-arm.

**Fix (v4):** Selective edge filtering before per-arm filleting:

```python
# H-arm: skip top-face edges in junction zone
def h_outer(e):
    m = edge_mid(e)
    return not (abs(m.y - junc_y) < 0.4 and abs(m.x) < arm_hx + 0.4)

# V-arm: skip bottom-face edges at Y=junc_y
def v_outer(e):
    m = edge_mid(e)
    return not (abs(m.y - junc_y) < 0.4)
```

After fusing, the inside junction corner edges (at X=±arm_hx, Y=junc_y, running Z=0→8) are **sharp 90°** because:
1. They didn't exist as edges before the fuse (created by fuse operation)
2. Neither adjacent face was rounded in the junction zone

### Full Python Code (v4)

```python
import FreeCAD, Part

doc = FreeCAD.ActiveDocument
for o in list(doc.Objects):
    doc.removeObject(o.Name)

pcb_t   = 1.5
wall    = 2.0
depth   = 8.0
stop_w  = 2.0
slot_d  = depth - stop_w    # 6.0
h_len   = 20.0
v_len   = 10.0
arm_w   = 2*wall + pcb_t    # 5.5
arm_hx  = arm_w / 2         # 2.75
junc_y  = pcb_t + wall      # 3.5
fillet_r = 1.2
rh, rw  = 0.3, 0.5

# Step 1: Raw T-body (try whole-T fillet first, fall back to per-arm)
hraw = Part.makeBox(h_len, arm_w, depth, FreeCAD.Vector(-h_len/2, -wall, 0))
vraw = Part.makeBox(arm_w, v_len, depth, FreeCAD.Vector(-arm_hx, junc_y, 0))
t_raw = hraw.fuse(vraw)

def edge_mid(e):
    pts = [v.Point for v in e.Vertexes]
    n = len(pts)
    return FreeCAD.Vector(sum(p.x for p in pts)/n,
                          sum(p.y for p in pts)/n,
                          sum(p.z for p in pts)/n)

def is_inside_junction(e):
    m = edge_mid(e)
    return abs(abs(m.x) - arm_hx) < 0.4 and abs(m.y - junc_y) < 0.4

outer_edges = [e for e in t_raw.Edges if not is_inside_junction(e)]

t_body = t_raw
fillet_ok = False
for r in [fillet_r, 0.9, 0.6]:
    try:
        t_body = t_raw.makeFillet(r, outer_edges)
        fillet_ok = True
        break
    except: pass

if not fillet_ok:
    # Fallback: fillet arms independently, skip junction-facing edges
    def h_outer(e):
        m = edge_mid(e)
        return not (abs(m.y - junc_y) < 0.4 and abs(m.x) < arm_hx + 0.4)

    def v_outer(e):
        m = edge_mid(e)
        return not (abs(m.y - junc_y) < 0.4)

    hbody = hraw
    vbody = vraw
    for r in [fillet_r, 0.9, 0.6]:
        try:
            hbody = hraw.makeFillet(r, [e for e in hraw.Edges if h_outer(e)])
            break
        except: pass
    for r in [fillet_r, 0.9, 0.6]:
        try:
            vbody = vraw.makeFillet(r, [e for e in vraw.Edges if v_outer(e)])
            break
        except: pass
    t_body = hbody.fuse(vbody)

# Step 2: Cut slots
hslot = Part.makeBox(h_len+2, pcb_t, slot_d, FreeCAD.Vector(-h_len/2-1, 0, 0))
t_body = t_body.cut(hslot)

vslot = Part.makeBox(pcb_t, wall+v_len+1, slot_d, FreeCAD.Vector(-pcb_t/2, pcb_t, 0))
t_body = t_body.cut(vslot)

# Step 3: Grip ridges
for xp in [-6.0, 0.0, 6.0]:
    t_body = t_body.fuse(Part.makeBox(rw, rh, slot_d, FreeCAD.Vector(xp-rw/2, 0.0, 0.0)))
for xp in [-6.0, 6.0]:
    t_body = t_body.fuse(Part.makeBox(rw, rh, slot_d, FreeCAD.Vector(xp-rw/2, pcb_t-rh, 0.0)))
for yp in [junc_y+2.0, junc_y+5.0, junc_y+8.0]:
    t_body = t_body.fuse(Part.makeBox(rh, rw, slot_d, FreeCAD.Vector(-pcb_t/2, yp-rw/2, 0.0)))
    t_body = t_body.fuse(Part.makeBox(rh, rw, slot_d, FreeCAD.Vector(pcb_t/2-rh, yp-rw/2, 0.0)))

obj = doc.addObject("Part::Feature", "T_Clip_v4")
obj.Shape = t_body
doc.recompute()
```

---

## Issues Encountered & Solutions

| Issue | Cause | Fix |
|-------|-------|-----|
| Bridge lifting vertical PCB | V-slot started above Y=pcb_t, leaving bridging material | Start V-slot at Y=pcb_t (exact top of base PCB) |
| Concave junction corners (v3) | Filleting all edges of both arms before fusing rounds junction zone | Skip junction-facing edges in fillet filter (v4) |
| BRep_API fillet failure | OCC can't fillet full T-body at once | Per-arm selective fillet as fallback |
| Chamfer failing after ridges | Ridge fuse changes topology | Apply chamfer BEFORE adding ridges |
| Screenshot token overflow | Default resolution too large | Use width=600, height=400 |

---

## Files in This Repo

| File | Description |
|------|-------------|
| `t-clip.FCStd` | FreeCAD project file (active document with T_Clip_v4) |
| `t-clip.20260408-004233.FCBak` | FreeCAD auto-backup |
| `t-clip-T_Clip_v4.stl` | STL export for 3D printing |
| `t-clip-T_Clip_v4.gcode.3mf` | Sliced print file (3MF format) |

---

## FreeCAD MCP Tool Notes

- **Tool**: `mcp__freecad__execute_code` — runs arbitrary Python in FreeCAD context
- **Screenshots**: `mcp__freecad__get_view` with `width=600, height=400` to avoid base64 token overload
- **Active document**: Always check `FreeCAD.ActiveDocument` before executing
- **OCC topology**: Fillet on fused bodies often fails — prefer per-component filleting
- **Edge filtering**: Use midpoint coordinates to identify and skip specific edges

---

*Session date: 2026-04-08*
