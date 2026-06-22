# Database container (MongoDB) — Webcam Filters App

This folder defines the **stable MongoDB contract** (collection names, document shapes, and indexes) used by the backend container.

> Scope: we persist **filter presets** and **snapshot metadata** (metadata only; not image binary payloads).

---

## Collections

### 1) `presets`

Stores reusable filter configurations saved by the user.

#### Document shape

```js
{
  _id: ObjectId,

  // Stable identifier for external references (e.g., frontend/back-end URLs).
  // A UUID string is recommended. Must be unique.
  presetId: "0f1f2d9b-....",

  name: "Warm cinematic",
  description: "Slightly warmer with more contrast",

  // Filter configuration the frontend can apply deterministically.
  // `filters` is an object keyed by filter name, values are normalized numbers.
  // Backend should validate ranges (typically 0..1 or -1..1 depending on filter).
  filters: {
    brightness: 1.05,  // example: 0..2
    contrast: 1.1,     // example: 0..2
    saturate: 1.2,     // example: 0..3
    grayscale: 0.0,    // 0..1
    sepia: 0.15,       // 0..1
    hueRotate: 10,     // degrees, e.g. -180..180
    blur: 0.0          // px, e.g. 0..20
  },

  // Optional tags for searching / grouping
  tags: ["cinematic", "warm"],

  // Soft delete support (optional)
  isDeleted: false,

  createdAt: ISODate("2026-06-22T00:00:00.000Z"),
  updatedAt: ISODate("2026-06-22T00:00:00.000Z")
}
```

#### Indexes

Create these indexes (names are suggestions; backend should assume the uniqueness semantics):

- Unique: `presetId`
- Search: `name` (and optionally `tags`)
- Optional: `(isDeleted, updatedAt)` for listing active presets efficiently

Recommended commands (run in `mongosh`):

```js
db.presets.createIndex({ presetId: 1 }, { unique: true, name: "uniq_presetId" });
db.presets.createIndex({ name: 1 }, { name: "idx_name" });
db.presets.createIndex({ tags: 1 }, { name: "idx_tags" });
db.presets.createIndex({ isDeleted: 1, updatedAt: -1 }, { name: "idx_active_recent" });
```

---

### 2) `snapshots`

Stores metadata about captured snapshots. Images are **not** stored in MongoDB for MVP; the client downloads locally.

#### Document shape

```js
{
  _id: ObjectId,

  // Stable identifier for external references. UUID string recommended.
  snapshotId: "a3c1...",

  // Optional linkage to preset used when snapshot was captured
  presetId: "0f1f2d9b-....", // may be null if custom filters were used

  // The exact filters used at capture time (denormalized for historical accuracy)
  filters: {
    brightness: 1.05,
    contrast: 1.1,
    saturate: 1.2,
    grayscale: 0.0,
    sepia: 0.15,
    hueRotate: 10,
    blur: 0.0
  },

  // Video/canvas capture context
  capture: {
    width: 1280,
    height: 720,
    mimeType: "image/png" // what the client generated
  },

  // Optional device info to help users differentiate cameras/sessions
  device: {
    deviceId: "browser-media-device-id",
    label: "FaceTime HD Camera"
  },

  createdAt: ISODate("2026-06-22T00:00:00.000Z")
}
```

#### Indexes

- Unique: `snapshotId`
- Listing: `createdAt` descending
- Optional: `presetId` for “show snapshots by preset”

Recommended commands:

```js
db.snapshots.createIndex({ snapshotId: 1 }, { unique: true, name: "uniq_snapshotId" });
db.snapshots.createIndex({ createdAt: -1 }, { name: "idx_createdAt_desc" });
db.snapshots.createIndex({ presetId: 1, createdAt: -1 }, { name: "idx_preset_createdAt" });
```

---

## Initialization / Seed Notes

This container **does not auto-run init scripts** by default. The backend should be able to start against an empty database and create records as needed, but the following should be done once per environment:

1. Ensure the database exists (MongoDB will create it on first write).
2. Create the indexes above for `presets` and `snapshots`.
3. (Optional) Seed a small set of default presets.

### Optional seed data (manual)

Example `insertMany` for a few presets:

```js
db.presets.insertMany([
  {
    presetId: "00000000-0000-0000-0000-000000000001",
    name: "Neutral",
    description: "No visible changes; useful baseline",
    filters: { brightness: 1, contrast: 1, saturate: 1, grayscale: 0, sepia: 0, hueRotate: 0, blur: 0 },
    tags: ["default"],
    isDeleted: false,
    createdAt: new Date(),
    updatedAt: new Date()
  },
  {
    presetId: "00000000-0000-0000-0000-000000000002",
    name: "Warm",
    description: "Slight warmth and saturation",
    filters: { brightness: 1.05, contrast: 1.05, saturate: 1.15, grayscale: 0, sepia: 0.12, hueRotate: 5, blur: 0 },
    tags: ["default", "warm"],
    isDeleted: false,
    createdAt: new Date(),
    updatedAt: new Date()
  }
]);
```

---

## Backend contract expectations (important)

- Collection names are **exactly**: `presets` and `snapshots`.
- External identifiers are **strings** (`presetId`, `snapshotId`) and are **unique**.
- Timestamps:
  - `createdAt` always present.
  - `updatedAt` present on presets.
- Snapshot persistence is **metadata-only** for MVP (no base64 image blobs).

If future requirements add actual image storage, prefer object storage (S3/GCS/etc.) and store only a URL + metadata in `snapshots`.

