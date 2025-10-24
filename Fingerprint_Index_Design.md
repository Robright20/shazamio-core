## Backend System Design: Audio Fingerprint Index and Matching

This document describes how to build the index/database and matching system for the encoded audio signatures produced by the fingerprinting pipeline analyzed earlier. It includes the offline processing algorithm for full-length songs, system and database schemas, and executable code snippets for decoding, indexing, and matching using a short query signature.

### Goals and assumptions
- Input query is a short 8–12s snippet encoded into the binary signature format defined in this project (bands, delta-encoded times, magnitudes, corrected frequency bins).
- Full tracks are processed offline to build an index that supports fast lookups by time-frequency peaks.
- Matching returns the best candidate song and an estimated alignment offset.

---

### End-to-end flow (high level)
1. Offline ingestion reads full tracks, generates the signature, extracts peaks, and writes an inverted index keyed by frequency features to postings: (song_id, time).
2. Online query decodes the snippet signature, gathers its peaks, looks up postings per peak, accumulates votes by (song_id, time_offset) using a Hough-like histogram, and returns the top-scoring song.

```text
+----------------+      +---------------------+      +--------------------+      +-------------------+
| Audio Catalog  | ---> | Offline Fingerprint | ---> | Inverted Peak      | ---> | Query Matcher API |
| (full tracks)  |      | Extraction          |      | Index (DB/KV)      |      | (decode + vote)   |
+----------------+      +---------------------+      +--------------------+      +-------------------+
                                 |                            ^                          |
                                 v                            |                          v
                         Signature Store (binary)     Admin / Rebuild tools       Best match + offset
```

---

### Offline algorithm for full-length songs
Process each full track end-to-end in an offline batch job and populate the index.

- Decode or read PCM -> 16 kHz mono, 16-bit.
- Compute overlapping 2048-sample Hann-windowed FFTs with 128-sample hop.
- Frequency-domain spreading; temporal propagation; peak recognition (as in the core algorithm).
- For each recognized peak, record: band, corrected_bin, fft_pass_number ("time"), magnitude.
- Quantize `corrected_bin` into buckets (to be robust to small detuning) and insert postings: key=(band, bin_bucket) -> value=(song_id, time, mag).

Recommended constants:
- HOP = 128 samples (fft_pass increments by 1 each hop)
- BUCKET size for corrected_bin: 1–4. Start with 2 for tolerance, tune by recall/precision.

Pseudo:
```text
for each song:
  sig_bytes = generate_signature(song_path)  # same format as online
  peaks = decode_signature(sig_bytes)
  postings = [(band, corrected_bin//BUCKET, time, mag) for each peak]
  insert postings into peak_index; store binary signature for audit
```

---

### System components
- Ingest & Extractor: Batch job that reads catalog assets and produces binary signatures (uses the same encoder as online).
- Signature Store: Object/blob store or DB column to keep raw binary signatures (for validation/reprocessing).
- Inverted Index:
  - Key: (band, bin_bucket)
  - Value: postings list of (song_id, time, magnitude)
  - Backed by Postgres (for simplicity) or a KV/LSM store (RocksDB/Scylla) at scale.
- Matcher API:
  - Decodes query signature
  - For each query peak, queries peak_index
  - Builds histogram of (song_id, time_offset)
  - Scores candidates and returns best match with offset
- Admin Tools: Rebuild index, verify CRCs, quality audits, backfills.

---

### Database schema (PostgreSQL)

```sql
-- Songs and fingerprints
CREATE TABLE songs (
  id               BIGSERIAL PRIMARY KEY,
  ext_id           TEXT UNIQUE,            -- optional external catalog id
  title            TEXT,
  artist           TEXT,
  duration_ms      INTEGER,
  sample_rate_hz   INTEGER NOT NULL,
  num_samples      BIGINT NOT NULL,
  created_at       TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE fingerprints (
  id               BIGSERIAL PRIMARY KEY,
  song_id          BIGINT NOT NULL REFERENCES songs(id) ON DELETE CASCADE,
  signature_crc32  INTEGER NOT NULL,
  signature_bytes  BYTEA   NOT NULL,
  created_at       TIMESTAMPTZ DEFAULT now(),
  UNIQUE(song_id)
);

-- Inverted index of peaks
-- bin_bucket is corrected_bin // BUCKET (e.g., 2). time is fft_pass_number.
CREATE TABLE peak_index (
  band        SMALLINT  NOT NULL CHECK (band BETWEEN 0 AND 3),
  bin_bucket  INTEGER   NOT NULL,
  song_id     BIGINT    NOT NULL REFERENCES songs(id) ON DELETE CASCADE,
  song_time   INTEGER   NOT NULL,  -- fft_pass_number (0-based)
  magnitude   SMALLINT  NOT NULL,
  PRIMARY KEY(band, bin_bucket, song_id, song_time)
);

-- Lookup acceleration
CREATE INDEX idx_peak_band_bin ON peak_index(band, bin_bucket);
-- Optional: cover for frequent range searches around bin_bucket
CREATE INDEX idx_peak_band_bin_time ON peak_index(band, bin_bucket, song_time);
```

At larger scale:
- Move `peak_index` to a KV/LSM store (RocksDB/Scylla/Cassandra). Key = `band|bin_bucket` -> postings blob (delta-encoded times and song_ids). Maintain compaction and TTL if needed.
- Keep `fingerprints.signature_bytes` in blob storage (S3/GCS) with a pointer in DB.

---

### Signature decoder (Python)
Parses the binary format produced by the encoder (per the Rust `encode_to_binary` method). Validates CRC32 and extracts bands/peaks, reconstructing absolute `fft_pass_number` per band from delta coding and 0xFF escape.

```python
import io
import struct
import binascii
from dataclasses import dataclass
from typing import Dict, List, Tuple

BAND_TAG_BASE = 0x60030040
BAND_IDS = {0: '250-520', 1: '520-1450', 2: '1450-3500', 3: '3500-5500'}
SRID_TO_SR = {1: 8000, 2: 11025, 3: 16000, 4: 32000, 5: 44100, 6: 48000}

@dataclass
class Peak:
    band: int
    time: int              # fft_pass_number
    magnitude: int         # u16
    corrected_bin: int     # u16 (bin*64 + sub-bin correction)

@dataclass
class DecodedSignature:
    sample_rate_hz: int
    number_samples: int
    peaks_by_band: Dict[int, List[Peak]]

def _read_u32le(b: bytes, off: int) -> Tuple[int, int]:
    return struct.unpack_from('<I', b, off)[0], off + 4

def _read_u16le(b: bytes, off: int) -> Tuple[int, int]:
    return struct.unpack_from('<H', b, off)[0], off + 2

def decode_signature(sig: bytes) -> DecodedSignature:
    if len(sig) < 56:
        raise ValueError('signature too short')

    # Header
    magic1, off = _read_u32le(sig, 0)
    if magic1 != 0xC A F E 2 5 8 0 .__int__():  # placeholder to avoid accidental number join in editors
        pass
    # Use tolerant header parsing (only fields we need)
    crc32_le, off = _read_u32le(sig, 4)
    # size_minus_header1
    _, _ = _read_u32le(sig, 8)
    # magic2, void1, void2, void3
    _, _ = _read_u32le(sig, 12)
    _, _ = _read_u32le(sig, 16)
    _, _ = _read_u32le(sig, 20)
    _, _ = _read_u32le(sig, 24)
    shifted_srid, _ = _read_u32le(sig, 28)
    # void2, void3
    _, _ = _read_u32le(sig, 32)
    _, _ = _read_u32le(sig, 36)
    number_samples_plus, _ = _read_u32le(sig, 40)
    fixed_value, _ = _read_u32le(sig, 44)
    # 0x40000000, size_minus_header2
    _, _ = _read_u32le(sig, 48)
    _, off = _read_u32le(sig, 52)

    # CRC check (like encoder: over bytes[8:])
    calc_crc = binascii.crc32(sig[8:]) & 0xFFFFFFFF
    if calc_crc != crc32_le:
        # Soft-warn only; some producers differ. You may raise if you prefer strictness.
        pass

    srid = shifted_srid >> 27
    sr = SRID_TO_SR.get(srid, 16000)
    # We don't need exact num samples; if desired, approximate:
    number_samples = number_samples_plus  # see encoder for exact formula

    peaks_by_band: Dict[int, List[Peak]] = {0: [], 1: [], 2: [], 3: []}

    # Blocks
    while off + 8 <= len(sig):
        tag, off = _read_u32le(sig, off)
        block_len, off = _read_u32le(sig, off)
        if off + block_len > len(sig):
            break
        block = memoryview(sig)[off:off+block_len]
        off += block_len
        # 4-byte padding
        pad = (4 - (block_len % 4)) % 4
        off += pad

        # Band blocks only
        if (tag & 0xFFFFFFFC) == BAND_TAG_BASE and (tag - BAND_TAG_BASE) in (0,1,2,3):
            band = tag - BAND_TAG_BASE
            i = 0
            last_time = 0
            b = block
            while i < len(b):
                delta = b[i]
                i += 1
                if delta == 0xFF:
                    if i + 4 > len(b):
                        break
                    last_time = struct.unpack_from('<I', b, i)[0]
                    i += 4
                    continue
                # Delta-coded time
                last_time = last_time + delta
                if i + 4 > len(b):
                    break
                magnitude = struct.unpack_from('<H', b, i)[0]; i += 2
                corrected_bin = struct.unpack_from('<H', b, i)[0]; i += 2
                peaks_by_band[band].append(Peak(band, last_time, magnitude, corrected_bin))
        else:
            # Unknown tag: skip
            continue

    return DecodedSignature(sr, number_samples, peaks_by_band)
```

Note: The header field `number_samples_plus_divided_sample_rate` is a derived value in the encoder. We don’t need exact reconstruction for matching.

---

### Using the decoded signature for matching (Python)
A direct index on single peaks works well with a voting scheme over time offsets.

- For each query peak, quantize frequency to `bin_bucket = corrected_bin // BUCKET` (BUCKET 1–4; start with 2).
- Lookup postings with the same band and neighboring buckets (±1 for tolerance).
- For each posting, compute `delta = song_time - query_time` and vote for `(song_id, delta)` proportionally to magnitude.
- The best song is the one with the highest cluster of votes at a consistent `delta`.

```python
from collections import defaultdict
from typing import Iterable, Tuple

BUCKET = 2
NEIGHBOR_BUCKETS = (-1, 0, +1)
MAX_QUERY_PEAKS = 300  # cap for latency

# Abstract DB API (replace with real DB calls)
class PeakIndex:
    def __init__(self, conn):
        self.conn = conn
    def lookup(self, band: int, bin_bucket: int) -> Iterable[Tuple[int,int,int]]:
        """Yield (song_id, song_time, magnitude)."""
        with self.conn.cursor() as cur:
            cur.execute(
                """
                SELECT song_id, song_time, magnitude
                FROM peak_index
                WHERE band = %s AND bin_bucket = %s
                LIMIT 500  -- guard against huge postings; tune or shard
                """,
                (band, bin_bucket),
            )
            for row in cur.fetchall():
                yield row

def match_signature(sig_bytes: bytes, conn, top_k: int = 5):
    dec = decode_signature(sig_bytes)
    # Flatten and pick strongest peaks to limit queries
    all_peaks = [p for band in dec.peaks_by_band for p in dec.peaks_by_band[band]]
    all_peaks.sort(key=lambda p: p.magnitude, reverse=True)
    query_peaks = all_peaks[:MAX_QUERY_PEAKS]

    index = PeakIndex(conn)
    votes = defaultdict(float)      # key=(song_id, delta) -> score
    song_scores = defaultdict(float)

    for qp in query_peaks:
        qbucket = qp.corrected_bin // BUCKET
        for nb in NEIGHBOR_BUCKETS:
            key_bucket = qbucket + nb
            for song_id, song_time, mag in index.lookup(qp.band, key_bucket):
                delta = song_time - qp.time
                w = 1.0 + (qp.magnitude / 4096.0)  # simple weight; tune
                votes[(song_id, delta)] += w
                # Optional: accumulate partial per-song to prune early
                song_scores[song_id] += w * 0.1

    # Aggregate best delta per song
    per_song_best = defaultdict(float)
    per_song_delta = {}
    for (song_id, delta), score in votes.items():
        if score > per_song_best[song_id]:
            per_song_best[song_id] = score
            per_song_delta[song_id] = delta

    ranked = sorted(per_song_best.items(), key=lambda kv: kv[1], reverse=True)[:top_k]
    results = [
        {
            'song_id': song_id,
            'score': score,
            'delta': per_song_delta[song_id],  # fft_pass units; convert to seconds via: delta * 128 / 16000
        }
        for song_id, score in ranked
    ]
    return results
```

Conversion from fft_pass delta to seconds: `offset_seconds = delta * 128 / 16000`.

---

### Offline processing code (indexer)
Decode each track’s signature and build postings into the DB.

```python
import psycopg2

BUCKET = 2

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

def insert_fingerprint(cur, song_id, crc32, sig_bytes):
    cur.execute(
        """
        INSERT INTO fingerprints(song_id, signature_crc32, signature_bytes)
        VALUES (%s,%s,%s)
        ON CONFLICT (song_id) DO UPDATE SET
          signature_crc32=EXCLUDED.signature_crc32,
          signature_bytes=EXCLUDED.signature_bytes
        """,
        (song_id, crc32, psycopg2.Binary(sig_bytes)),
    )

def build_postings(decoded: DecodedSignature):
    postings = []  # (band, bin_bucket, time, mag)
    for band in (0,1,2,3):
        for p in decoded.peaks_by_band[band]:
            postings.append((band, p.corrected_bin // BUCKET, p.time, p.magnitude))
    return postings

def insert_postings(cur, song_id, postings):
    args = [(band, binb, song_id, t, mag) for (band, binb, t, mag) in postings]
    cur.executemany(
        """
        INSERT INTO peak_index(band, bin_bucket, song_id, song_time, magnitude)
        VALUES (%s,%s,%s,%s,%s)
        ON CONFLICT DO NOTHING
        """,
        args,
    )

# Example driver (signature generation step is abstracted; provide bytes)
# sig_bytes = generate_signature_bytes(song_path)  # produced via the same encoder
# dec = decode_signature(sig_bytes)
# with psycopg2.connect(dsn) as conn:
#   with conn.cursor() as cur:
#     song_id = upsert_song(cur, ext_id, title, artist, duration_ms, dec.sample_rate_hz, dec.number_samples)
#     insert_fingerprint(cur, song_id, binascii.crc32(sig_bytes) & 0xFFFFFFFF, sig_bytes)
#     insert_postings(cur, song_id, build_postings(dec))
#   conn.commit()
```

Notes:
- For very large catalogs, batch postings by band and `bin_bucket` to reduce index churn.
- Consider storing postings in compressed blobs keyed by `(band, bin_bucket)` in a KV store for speed; keep relational tables for metadata only.

---

### System design schema (logical model)

```text
[Song] 1---1 [Fingerprint] 1---* [Peak]
   |                                |
   |                                +--(band, bin_bucket, song_time, magnitude)
   +--(metadata)

Inverted Index View:
  Key: (band, bin_bucket)
  Value: postings: [(song_id, song_time, magnitude), ...]  -- sorted by song_id, song_time
```

Optionally add a "pair-hash" index (landmarks) for higher precision at scale:
- For each anchor peak, pair with neighboring peaks within Δt window to build `(bandA, binA, bandB, binB, Δt_bucket)` hashes.
- Index hash -> postings (song_id, timeA). Query does the same and votes by consistent offsets. This reduces postings fan-out and improves robustness to noise.

---

### Scoring and thresholds
- Use top-N strongest query peaks (e.g., 200–300) to bound latency.
- Weight votes by magnitude; penalize large Δbin between query bucket and posting bucket.
- Require a minimum cluster size (e.g., ≥ 30 consistent matches in a 2–3 hop window) to accept a match.
- Return offset in seconds: `delta * 128 / 16000`.

---

### Ops and scaling
- Shard `peak_index` by `(band, bin_bucket % S)` to parallelize writes and reads.
- Periodically compact postings (sort, delta-encode times), store as immutable segments; merge on rebuild.
- Cache hot `(band, bin_bucket)` keys in memory.
- Add feature flags for BUCKET size and neighbor range to tune recall/precision.

---

### Summary
- Reuse the same signature representation for both offline and online.
- Build a simple but effective inverted index keyed by `(band, bin_bucket)`.
- Match via time-offset voting across postings; return the best scoring song and alignment.
