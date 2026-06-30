# SEE File Format Overview

**SEE** (*Situated Experience Encoding*) stores building or site geometry together with sensory observations made during an in-person inspection.

The format uses three file types:

- `.sem` — **Situated Experience Model**: static scene data generated from IFC, GeoJSON, or similar source data.
- `.sei` — **Situated Experience Investigation**: one inspection process, including observations, questions, conclusions, and reasoning results.
- `.npz` — voxel storage for **VolumeFields**, such as odor, sound and temperature.

The core distinction is:

- **SceneObjects** describe bounded things: walls, pipes, doors, slabs, beams, rooms, sensors, terrain objects.
- **VolumeFields** describe continuous spatial phenomena: smell, sound, temperature, humidity.

The two worlds are linked by their shared coordinate system, and a **VolumeField** called `substance`, which tracks which material occupies each grid cell.

During an inspection, sensory observations are stored either as object-bound **Impressions** or field-bound **FieldImpressions**.

---

## Package layout

```text
example_project/
  building.sem
  inspection_1.sei
  substance.npz
  thermal.npz
  olfactory_musty.npz
```

A `.sem` file can be reused by many `.sei` files. A `.sei` file references exactly one `.sem` file.

---

## Sensory mapping

| Sense | Stored as | Spatial representation | Example |
|---|---|---|---|
| `visual` | `Impression` | `SceneObject` | Crack in a wall |
| `acoustic` | `FieldImpression` | `VolumeField` | Dripping sound |
| `olfactory` | `FieldImpression` | `VolumeField` | Musty smell |
| `haptic_tactile` | `Impression` | `SceneObject` | Wet plaster |
| `haptic_kinesthetic` | `Impression` | `SceneObject` | Beam flexes when pushed |
| `haptic_thermal_surface` | `Impression` | `SceneObject` | Cold wall |
| `haptic_thermal_field` | `FieldImpression` | `VolumeField` | Warm air flow |

`FieldImpressions` have an `intensity` from `0.0` to `1.0` (low to high, derived from user descriptions) and can be interpolated into new `VolumeFields`. Each acoustic or olfactory field also has a category, describing the quality of the sound or smell. This category is appended to the name of the sense with a `-` when stored as an `.npz` file.

---

## `.sem` — Situated Experience Model

A `.sem` file stores the reusable scene model: coordinate system, shared voxel grid, scene objects, and references to voxel fields.

```json
{
  "see_format": "SEM",
  "format_version": "0.1",
  "id": "demo-building",
  "name": "Demo Building",
  "coordinate_reference": {
    "crs": "local",
    "grid_origin": [0.0, 0.0, 0.0],
    "grid_shape": [100, 80, 30],
    "cell_size": 0.1
  },
  "scene_objects": [
    {
      "id": 33,
      "name": "north wall",
      "type": "Wall",
      "material": "plaster on masonry",
      "vertices": [[1.0, 2.0, 0.0], [1.0, 4.0, 0.0], [1.0, 4.0, 3.0]],
      "faces": [[0, 1, 2]]
    }
  ],
  "volume_fields": [
    {
      "name": "substance",
      "file": "substance.npz"
    },
    {
      "name": "olfactory-musty",
      "file": "olfactory-musty.npz"
    }
  ]
}
```

---

## `.sei` — Situated Experience Investigation

A `.sei` file stores the inspection layer: observations, sensory impressions, and reasoning results.

```json
{
  "see_format": "SEI",
  "id": "inspection-1",
  "sem_ref": "building.sem",
  "theme": "Structural health monitoring",
  "created_at": "2026-06-30T09:00:00.000Z",
  "reference": [
    {
      "id": "ref-1",
      "name": "wall upper right corner",
      "coordinates": [0.9, 3.67, 2.8],
      "element_reference": 33,
      "impressions": [
        {
          "id": "imp-1",
          "sense": "haptic_tactile",
          "statement": "There is slight bulging in the plaster.",
          "created_at": "2026-06-30T09:15:00.000Z"
        }
      ],
      "questions": [],
      "conclusions": []
    }
  ],
  "field_impressions": [
    {
      "id": "field-imp-1",
      "sense": "olfactory",
      "coordinates": [1.0, 3.0, 1.4],
      "intensity": 0.5,
      "created_at": "2026-06-30T09:15:30.000Z",
      "description": "Musty smell near the north wall.",
      "field": "musty"
    }
  ]
}
```

---

## `.npz` — voxel field storage

Voxel fields are stored as compressed NumPy `.npz` files.

Required array:

```text
values[x_index, y_index, z_index]
```

The `values` array must have the same shape as `voxel_grid.shape` in the referenced `.sem` file. Each value is either a number from `0.0` to `1.0`, or in the case of the `susbtance` voxel field, a string derived from the `material` property of the SceneObject that occupies the center of the cell.

Example:

```python
import numpy as np

shape = (100, 80, 30)
values = np.zeros(shape, dtype=np.float32)
values[10, 30, 14] = 0.5

np.savez_compressed("olfactory-musty.npz", values=values)
```

---

## Validation

A SEE dataset is valid if:

1. The `.sem` file has `see_format: "SEM"`.
2. The `.sei` file has `see_format: "SEI"` and a valid `sem_ref`.
3. All coordinates use the `.sem` coordinate reference.
4. All `.npz` arrays match `voxel_grid.shape`.
5. The `.sem` file references one `.npz` file with the name `substance`. This array contains string values.
6. Every `element_reference` references an existing `SceneObject`.
7. Every `FieldImpression.intensity` is between `0.0` and `1.0`.
8. `Impression` uses object-bound senses; `FieldImpression` uses field-bound senses.

---

## TODO

- Include example datasets
- Handling of time and uncertainty, specifically for VolumeFields that can evolve over time
- Handling of sensor data

---

## Cite this work

```text
Berger, M., Hennecke, D. (2026). Encoding Situated Experience: Agent-Driven Capture and Investigation of Sensory Impressions during In-Person Site Inspections. In Proceedings of Workshop of the European Group for Intelligent Computing in Engineering. EG-ICE 2026.
```
