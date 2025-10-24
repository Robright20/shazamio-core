## Backend System Design: Landmark (Shazam-style) Index and Matching

This revision aligns the design with the Shazam paper (Wang, “An Industrial-Strength Audio Search Algorithm”), using landmark/peak-pair hashes instead of single-peak postings. It preserves the project’s fingerprint format (bands, corrected_bin, per-FFT time) and builds an index that is fast, noise-robust, and scalable.

Key premise: The query is a short snippet (≈10 s) whose signature contains a sparse set of salient peaks. We form anchor→target peak pairs inside a time–frequency target zone, quantize them into hashes, and index those hashes. At query time, we form the same hashes and vote for consistent time offsets per song.

### What this changes vs previous version
- Uses Shazam-style landmarks (anchor–target peak pairs) with a compact 32-bit hash.
- Inverted index key is the pair-hash (not a single peak). Values are (song_id, anchor_time).
- Matching accumulates votes in a (song_id, time_offset) histogram from pair matches, yielding sharp clusters for the correct song.

---

### Parameters (grounded in this repo’s fingerprint)
- Sample rate: 16 kHz
- Window size: 2048, hop: 128 samples (≈8 ms/frame)
- Frequency representation: `corrected_bin` in units of 1/64 of an FFT bin; nominal FFT bins are 0..1024.

Recommended landmark parameters (tunable):
- Target zone (relative to anchor time): Δt_frames ∈ [DT_MIN, DT_MAX] = [8, 400]  → ≈[64 ms, 3.2 s]
- Frequency pairing tolerance: allow any target peak; optionally restrict |f2 − f1| ≤ 512 FFT bins
- Quantization:
  - f_bucket = floor((corrected_bin/64) / F_BUCKET), with F_BUCKET = 3 bins (≈0.14% at 16 kHz)
  - dt_bucket = floor(Δt_frames / DT_BUCKET), with DT_BUCKET = 3 frames (≈24 ms)
- Hash layout (32-bit): H = (f1_bucket << 20) | (f2_bucket << 10) | dt_bucket (10/10/10 bits)
- Pair fanout cap per anchor: cap target pairs per anchor to MAX_TARGETS = 5–15 to bound index size

These are standard, robust defaults; adjust per quality/latency/size SLAs.

---

### End-to-end flow
1. Offline: For each song, decode/generate signature, extract peaks, build landmark hashes, and insert (hash → postings of (song_id, anchor_time)).
2. Online: Decode query signature, build its landmark hashes, look up postings, vote for (song_id, time_offset = anchor_time_song − anchor_time_query), and return top clustered result(s).

```text
+----------------+     +----------------------+     +-----------------------+     +-------------------+
| Audio Catalog  | --> | Offline Peak Pairs   | --> | Landmark Hash Index   | --> | Query Matcher API |
| (full tracks)  |     | (anchor→target)      |     |  hash → [(song,t1)..] |     |  (decode+vote)    |
+----------------+     +----------------------+     +-----------------------+     +-------------------+
                          |                                        ^                       |
                          v                                        |                       v
                  Signature Store (binary)                   Admin/Rebuild           Best song + offset
```

---

### System components
- Ingest/Extractor: produces binary signatures (for audit) and peak lists.
- Landmark Builder: forms anchor→target pairs, quantizes to 32-bit hashes, bulk-writes postings.
- Landmark Hash Index: persistent inverted index; optimized for key-based fanout lookups.
- Matcher API: decodes query signature, forms hashes, looks up postings, and scores candidates via a (song_id, Δt) histogram.
- Admin: backfill/rebuild tools, periodic compaction, quality dashboards.

---

### Database schema (PostgreSQL reference)
For production scale, a KV/LSM store (RocksDB/Scylla/Cassandra) is ideal, but Postgres works for correctness and moderate scale.

```sql
-- Song metadata
CREATE TABLE songs (
  id               BIGSERIAL PRIMARY KEY,
  ext_id           TEXT UNIQUE,
  title            TEXT,
  artist           TEXT,
  duration_ms      INTEGER,
  sample_rate_hz   INTEGER NOT NULL,
  num_samples      BIGINT  NOT NULL,
  created_at       TIMESTAMPTZ DEFAULT now()
);

-- Raw binary fingerprint (for audit/reprocessing)
CREATE TABLE fingerprints (
  song_id          BIGINT  PRIMARY KEY REFERENCES songs(id) ON DELETE CASCADE,
  signature_crc32  INTEGER NOT NULL,
  signature_bytes  BYTEA   NOT NULL,
  created_at       TIMESTAMPTZ DEFAULT now()
);

-- Shazam-style landmark index: hash -> postings
-- hash_key packs (f1_bucket, f2_bucket, dt_bucket) into 32 bits
CREATE TABLE hash_index (
  hash_key   INTEGER  NOT NULL,
  song_id    BIGINT   NOT NULL REFERENCES songs(id) ON DELETE CASCADE,
  t_anchor   INTEGER  NOT NULL,   -- anchor time in frames (hop units = 128 samples)
  PRIMARY KEY (hash_key, song_id, t_anchor)
);

-- Fast lookup by hash
CREATE INDEX idx_hash_key ON hash_index(hash_key);

-- Optional diagnostics / compaction aids
CREATE MATERIALIZED VIEW hash_stats AS
SELECT hash_key, count(*) AS postings
FROM hash_index
GROUP BY hash_key;
```

KV/LSM variant: key=`hash_key`, value=sorted postings blob with delta-encoded `song_id` and `t_anchor`, capped by max fanout per key.

---

### Algorithms

- Offline (per-song):
  - Decode/generate signature
  - Extract peaks (already done by the repo’s algorithm); flatten by time
  - For each anchor peak at time t1, pair with target peaks at times t2 ∈ [t1+DT_MIN, t1+DT_MAX]
  - Quantize (f1, f2, Δt) → (f1_bucket, f2_bucket, dt_bucket), pack into `hash_key`
  - Insert (hash_key, song_id, t_anchor=t1)

- Online (query-time):
  - Decode query signature, extract peaks
  - Build query landmark hashes with same quantization
  - For each query (hash_key, t1_q), fetch postings [(song_id, t1_s)]
  - Compute Δt = t1_s − t1_q; vote in histogram[(song_id, Δt)] += weight
  - Pick top cluster by count/score in a small Δt window; return best song and offset

---

### Signature decoder (Python)
Parses the project’s binary format to (band, time, magnitude, corrected_bin). CRC is checked but non-fatal.

```python
import struct
import binascii
from dataclasses import dataclass
from typing import Dict, List, Tuple

BAND_TAG_BASE = 0x60030040
SRID_TO_SR = {1: 8000, 2: 11025, 3: 16000, 4: 32000, 5: 44100, 6: 48000}

@dataclass
class Peak:
    band: int
    time: int              # fft_pass_number (frames of 128 samples)
    magnitude: int         # u16 (scaled log magnitude)
    corrected_bin: int     # u16, bin*64 + sub-bin correction

@dataclass
class DecodedSignature:
    sample_rate_hz: int
    number_samples: int
    peaks_by_band: Dict[int, List[Peak]]

def _u32(b: bytes, off: int) -> Tuple[int, int]:
    return struct.unpack_from('<I', b, off)[0], off + 4

def _u16(b: bytes, off: int) -> Tuple[int, int]:
    return struct.unpack_from('<H', b, off)[0], off + 2

def decode_signature(sig: bytes) -> DecodedSignature:
    if len(sig) < 56:
        raise ValueError('signature too short')
    magic1, off = _u32(sig, 0)
    # Optional strictness: assert magic1 == 0xCAFE2580
    crc32_le, _ = _u32(sig, 4)
    # Skip to fields we need
    shifted_srid, _ = _u32(sig, 28)
    number_samples_plus, _ = _u32(sig, 40)
    # Header tail
    _, off = _u32(sig, 52)

    # Verify CRC over payload (bytes[8:])
    calc_crc = binascii.crc32(sig[8:]) & 0xFFFFFFFF
    # Non-fatal if mismatch (producers may differ slightly)
    _ = (calc_crc == crc32_le)

    srid = shifted_srid >> 27
    sr = SRID_TO_SR.get(srid, 16000)
    # The encoder wrote: number_samples + int(sr * 0.24)
    number_samples = max(0, number_samples_plus - int(sr * 0.24))

    peaks_by_band: Dict[int, List[Peak]] = {0: [], 1: [], 2: [], 3: []}

    while off + 8 <= len(sig):
        tag, off = _u32(sig, off)
        block_len, off = _u32(sig, off)
        if off + block_len > len(sig):
            break
        block = memoryview(sig)[off:off + block_len]
        off += block_len
        off += (4 - (block_len % 4)) % 4   # 4-byte padding

        band = tag - BAND_TAG_BASE
        if (tag & 0xFFFFFFFC) != BAND_TAG_BASE or band not in (0, 1, 2, 3):
            continue

        i = 0
        last_time = 0
        while i < len(block):
            delta = block[i]
            i += 1
            if delta == 0xFF:
                if i + 4 > len(block):
                    break
                last_time = struct.unpack_from('<I', block, i)[0]
                i += 4
                continue
            last_time += delta
            if i + 4 > len(block):
                break
            magnitude = struct.unpack_from('<H', block, i)[0]; i += 2
            corrected_bin = struct.unpack_from('<H', block, i)[0]; i += 2
            peaks_by_band[band].append(Peak(band, last_time, magnitude, corrected_bin))

    return DecodedSignature(sr, number_samples, peaks_by_band)
```

---

### Landmark (pair) generation and hashing (Python)
Build landmark hashes from decoded peaks.

```python
from typing import Iterable, List, Tuple

# Tunables (see Parameters section)
F_BUCKET = 3           # FFT bins per bucket
DT_BUCKET = 3         # frames per bucket
DT_MIN = 8            # frames (~64 ms)
DT_MAX = 400          # frames (~3.2 s)
MAX_TARGETS = 10      # cap target pairs per anchor

def to_fft_bin(corrected_bin: int) -> int:
    return corrected_bin // 64

def bucket_f(bin_i: int) -> int:
    return bin_i // F_BUCKET

def bucket_dt(dt_frames: int) -> int:
    return dt_frames // DT_BUCKET

def pack_hash(f1b: int, f2b: int, dtb: int) -> int:
    # 10 bits each (fits practical ranges); mask to be safe
    return ((f1b & 0x3FF) << 20) | ((f2b & 0x3FF) << 10) | (dtb & 0x3FF)

def peaks_flat(dec) -> List[Tuple[int,int,int]]:
    # Return [(time, fft_bin, magnitude)], across all bands
    flat = []
    for band in (0,1,2,3):
        for p in dec.peaks_by_band[band]:
            flat.append((p.time, to_fft_bin(p.corrected_bin), p.magnitude))
    flat.sort(key=lambda x: x[0])
    return flat

def make_landmarks(dec) -> List[Tuple[int,int]]:
    """Return [(hash_key, t_anchor)] for a decoded signature."""
    pts = peaks_flat(dec)
    L: List[Tuple[int,int]] = []
    n = len(pts)
    j_start = 0
    for i in range(n):
        t1, f1, _ = pts[i]
        # Advance j_start to first t2 >= t1 + DT_MIN
        while j_start < n and pts[j_start][0] < t1 + DT_MIN:
            j_start += 1
        cnt = 0
        j = j_start
        while j < n and pts[j][0] <= t1 + DT_MAX and cnt < MAX_TARGETS:
            t2, f2, _ = pts[j]
            dt = t2 - t1
            if dt >= DT_MIN:
                h = pack_hash(bucket_f(f1), bucket_f(f2), bucket_dt(dt))
                L.append((h, t1))
                cnt += 1
            j += 1
    return L
```

---

### Offline indexing (Python)
Upserts song metadata, stores raw signature, and inserts landmark postings.

```python
import psycopg2, binascii
from typing import Iterable, Tuple

def upsert_song(cur, ext_id, title, artist, duration_ms, sample_rate_hz, num_samples) -> int:
    cur.execute(
        """
        INSERT INTO songs(ext_id, title, artist, duration_ms, sample_rate_hz, num_samples)
        VALUES (%s,%s,%s,%s,%s,%s)
        ON CONFLICT (ext_id) DO UPDATE SET
          title=EXCLUDED.title,
          artist=EXCLUDED.artist,
          duration_ms=EXCLUDED.duration_ms,
          sample_rate_hz=EXCLUDED.sample_rate_hz,
          num_samples=EXCLUDED.num_samples
        RETURNING id
        """,
        (ext_id, title, artist, duration_ms, sample_rate_hz, num_samples),
    )
    return cur.fetchone()[0]

def insert_fingerprint(cur, song_id, sig_bytes: bytes):
    cur.execute(
        """
        INSERT INTO fingerprints(song_id, signature_crc32, signature_bytes)
        VALUES (%s, %s, %s)
        ON CONFLICT (song_id) DO UPDATE SET
          signature_crc32 = EXCLUDED.signature_crc32,
          signature_bytes = EXCLUDED.signature_bytes
        """,
        (song_id, binascii.crc32(sig_bytes) & 0xFFFFFFFF, psycopg2.Binary(sig_bytes)),
    )

def insert_landmarks(cur, song_id: int, landmarks: Iterable[Tuple[int,int]]):
    cur.executemany(
        """
        INSERT INTO hash_index(hash_key, song_id, t_anchor)
        VALUES (%s, %s, %s)
        ON CONFLICT DO NOTHING
        """,
        [(h, song_id, t) for (h, t) in landmarks],
    )

# Driver (signature production omitted if you already have bytes)
# sig_bytes = generate_signature_bytes(song_path)
# dec = decode_signature(sig_bytes)
# landmarks = make_landmarks(dec)
# with psycopg2.connect(dsn) as conn:
#   with conn.cursor() as cur:
#     song_id = upsert_song(cur, ext_id, title, artist, duration_ms, dec.sample_rate_hz, dec.number_samples)
#     insert_fingerprint(cur, song_id, sig_bytes)
#     insert_landmarks(cur, song_id, landmarks)
#   conn.commit()
```

---

### Query-time matching (Python)
Form query landmarks, look up postings by hash, vote for consistent time offsets per song, and return the best cluster.

```python
from collections import defaultdict
from typing import List, Dict, Tuple

MAX_HASH_HITS = 500  # guardrail per hash to bound fanout

class HashIndex:
    def __init__(self, conn):
        self.conn = conn
    def lookup(self, hash_key: int) -> List[Tuple[int,int]]:
        with self.conn.cursor() as cur:
            cur.execute(
                """
                SELECT song_id, t_anchor
                FROM hash_index
                WHERE hash_key = %s
                LIMIT %s
                """,
                (hash_key, MAX_HASH_HITS),
            )
            return cur.fetchall()

def match_signature(sig_bytes: bytes, conn, top_k: int = 5):
    dec = decode_signature(sig_bytes)
    landmarks = make_landmarks(dec)
    index = HashIndex(conn)

    votes: Dict[Tuple[int,int], int] = defaultdict(int)  # (song_id, delta) -> count
    per_song_peak = defaultdict(int)

    for h, t1_q in landmarks:
        for song_id, t1_s in index.lookup(h):
            delta = t1_s - t1_q
            votes[(song_id, delta)] += 1
            per_song_peak[song_id] = max(per_song_peak[song_id], votes[(song_id, delta)])

    # Pick best delta per song and rank by votes
    per_song_best: Dict[int, Tuple[int,int]] = {}  # song_id -> (delta, votes)
    for (song_id, delta), v in votes.items():
        if song_id not in per_song_best or v > per_song_best[song_id][1]:
            per_song_best[song_id] = (delta, v)

    ranked = sorted(per_song_best.items(), key=lambda kv: kv[1][1], reverse=True)[:top_k]
    results = [
        {
            'song_id': sid,
            'votes': v,
            'offset_frames': d,
            'offset_seconds': d * 128 / 16000.0,
        }
        for sid, (d, v) in ranked
    ]
    return results
```

---

### Practical considerations from the paper (and field experience)
- Index size management: cap MAX_TARGETS per anchor and MAX_HASH_HITS per key to avoid heavy fanout; optionally keep only the first K postings per hash per song (tf-idf-like pruning).
- Robustness: the pair-hash is tolerant to small pitch/time perturbations due to quantization buckets and Δt discretization.
- Speed: queries touch only a few hundred hashes; lookups and histogramming are O(hits). Sharding by `hash_key % S` scales linearly.
- Quality: enforce a minimum vote threshold and require a tight cluster around the top Δt (e.g., ±2 buckets) before accepting a match.

---

### Summary
- Updated to Shazam-style landmark hashing for indexing and matching.
- Provided database schema, offline/online algorithms, and Python code to decode, index, and match signatures.
- Parameters are chosen to match the project’s FFT/hop configuration and can be tuned for precision/recall/latency/size.
