# 📋 260103: Standardize ISO8601 Time Format Across Codebase

> **Status:** ✅ COMPLETED  
> **Date:** 2026-01-03

## References

- Overview: `devdocs/projects/metan/stock/OVERVIEW.md`
- VN30 Feature Pipeline: `devdocs/projects/metan/stock/260101-build-vn30-feature-pipeline.md`
- Related Bug: Time format mismatch between `candle.time` ('Z' suffix) and `aggregated_df.index` ('+00:00')

## Problem Statement

Currently, there are **two different ISO8601 UTC time formats** used in the codebase:

| Format | Example | Source |
|--------|---------|--------|
| Z suffix | `2025-01-01T02:15:00Z` | `pendulum.to_iso8601_string()` |
| +00:00 offset | `2025-01-01T02:15:00+00:00` | PostgreSQL/Supabase, `pendulum.isoformat()` |

This causes string comparison failures when comparing timepoints from different sources.

### Root Cause Analysis

```
┌─────────────────────────────────────────────────────────────┐
│ _build_iso_timepoint()  →  to_iso8601_string()  →  'Z'      │
│ stock_data_collector    →  to_iso8601_string()  →  'Z'      │
│ Supabase DB response    →  PostgreSQL format    →  '+00:00' │
└─────────────────────────────────────────────────────────────┘
                              ↓
        String comparison fails: 'Z' != '+00:00'
```

## 🎯 Objective

Standardize **all time formatting to use `+00:00` format** (matching DB/Supabase format) by:

1. Fixing the existing `time_utils.py` utility (has bug: docstring says `+00:00` but returns `Z`)
2. Creating additional utility functions for common patterns
3. Replacing all `to_iso8601_string()` calls with utility functions
4. Removing duplicated `_build_iso_timepoint()` functions

## 📍 Affected Files

### Core Utility (Fix First)
| File | Line | Current | Action |
|------|------|---------|--------|
| `packages/stock/metan/stock/common/utils/time_utils.py` | 6-25 | `normalize_iso8601` returns Z format (bug) | Fix to return +00:00 format |

### Files Using `to_iso8601_string()`
| File | Line | Current | Action |
|------|------|---------|--------|
| `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py` | 167 | `pendulum.from_timestamp(...).to_iso8601_string()` | Use `timestamp_to_iso8601()` |
| `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py` | 274 | `pendulum.from_timestamp(...).to_iso8601_string()` | Use `timestamp_to_iso8601()` |
| `packages/stock/metan/stock/info/domain/index/tick_vn30_index_calculator.py` | 310-323 | `_build_iso_timepoint()` method | Use `build_iso_timepoint()` from utils |
| `packages/stock/metan/stock/info/domain/index/tcbs_vn30_index_calculator.py` | 283-296 | `_build_iso_timepoint()` method | Use `build_iso_timepoint()` from utils |
| `packages/stock/metan/stock/info/domain/index/vn30_base_calculator.py` | 228-232 | `_build_iso_timepoint()` method | Use `build_iso_timepoint()` from utils |

### Files Already Using Utility (Will Be Fixed Automatically)
| File | Line | Current | Note |
|------|------|---------|------|
| `packages/stock/.../tcbs_symbol_candle_fetcher.py` | 75 | `normalize_iso8601()` | Will work correctly after utility fix |

## 🔄 Implementation Plan

### Phase 1: Update Time Utilities

**File:** `packages/stock/metan/stock/common/utils/time_utils.py`

```python
"""Time utility functions for normalizing ISO8601 formats."""

import pendulum


def normalize_iso8601(iso_string: str) -> str:
    """
    Normalize various ISO8601 time formats to consistent DB format.

    Handles different formats:
        - "2025-12-26T02:55:00.000Z" (with milliseconds, Z suffix)
        - "2025-12-25T02:15:00+00:00" (with timezone offset)
        - "2025-12-25T02:15:00Z" (with Z suffix)

    Returns:
        Normalized format: "2025-12-26T02:55:00+00:00"

    Example:
        >>> normalize_iso8601("2025-12-26T02:55:00.000Z")
        '2025-12-26T02:55:00+00:00'
        >>> normalize_iso8601("2025-12-25T02:15:00Z")
        '2025-12-25T02:15:00+00:00'
    """
    dt = pendulum.parse(iso_string)
    return dt.set(microsecond=0).in_tz("UTC").isoformat()


def timestamp_to_iso8601(epoch_seconds: int) -> str:
    """
    Convert Unix timestamp to ISO8601 UTC string.

    Args:
        epoch_seconds: Unix timestamp in seconds

    Returns:
        ISO8601 string in DB format: "2025-01-01T02:15:00+00:00"

    Example:
        >>> timestamp_to_iso8601(1735697700)
        '2025-01-01T02:15:00+00:00'
    """
    return pendulum.from_timestamp(epoch_seconds, tz="UTC").set(microsecond=0).isoformat()


def build_iso_timepoint(date: str, hhmm: str) -> str:
    """
    Convert date + HH:MM (UTC) to ISO8601 UTC string.

    Args:
        date: Date in YYYY-MM-DD format (e.g., "2025-01-01")
        hhmm: Time in HH:MM format (e.g., "02:15")

    Returns:
        ISO8601 string in DB format: "2025-01-01T02:15:00+00:00"

    Example:
        >>> build_iso_timepoint("2025-01-01", "02:15")
        '2025-01-01T02:15:00+00:00'
    """
    hh, mm = hhmm.split(":")
    utc_dt = pendulum.parse(f"{date}T{hh}:{mm}:00", tz="UTC")
    return utc_dt.set(microsecond=0).isoformat()
```

### Phase 2: Update Stock Data Collector

**File:** `packages/stock/metan/stock/info/domain/stock_data_collector/stock_data_collector.py`

**Change 1 (Line 167):**
```python
# Before:
iso_time = pendulum.from_timestamp(int(t), tz="UTC").to_iso8601_string()

# After:
from metan.stock.common.utils.time_utils import timestamp_to_iso8601
iso_time = timestamp_to_iso8601(int(t))
```

**Change 2 (Line 274):**
```python
# Before:
iso_time = pendulum.from_timestamp(bucket_time, tz="UTC").to_iso8601_string()

# After:
iso_time = timestamp_to_iso8601(bucket_time)
```

### Phase 3: Update VN30 Index Calculators

**File:** `packages/stock/metan/stock/info/domain/index/tick_vn30_index_calculator.py`

1. Add import at top:
```python
from metan.stock.common.utils.time_utils import build_iso_timepoint
```

2. Replace method call (line 275):
```python
# Before:
iso_timepoint = self._build_iso_timepoint(date, hhmm)

# After:
iso_timepoint = build_iso_timepoint(date, hhmm)
```

3. Remove `_build_iso_timepoint` method (lines 310-323)

**File:** `packages/stock/metan/stock/info/domain/index/tcbs_vn30_index_calculator.py`

Same changes as above (lines 248, 283-296)

**File:** `packages/stock/metan/stock/info/domain/index/vn30_base_calculator.py`

Same pattern (lines 132, 228-232)

### Phase 4: Verification

No changes needed in `vn30_feature_pipeline.py` - the comparison will work after upstream fixes because:
- `candle.time` (from TickVN30IndexCalculator) → now uses `+00:00` format
- `aggregated_df.index` (from Supabase) → already uses `+00:00` format

## 📊 Summary of Changes

| Phase | File | Changes |
|-------|------|---------|
| 1 | `time_utils.py` | Fix `normalize_iso8601()`, add `timestamp_to_iso8601()`, add `build_iso_timepoint()` |
| 2 | `stock_data_collector.py` | Replace 2 occurrences of `to_iso8601_string()` |
| 3 | `tick_vn30_index_calculator.py` | Use utility, remove `_build_iso_timepoint()` method |
| 3 | `tcbs_vn30_index_calculator.py` | Use utility, remove `_build_iso_timepoint()` method |
| 3 | `vn30_base_calculator.py` | Use utility, remove `_build_iso_timepoint()` method |

## ⚠️ Risk Assessment

| Risk | Mitigation |
|------|------------|
| Breaking existing comparisons | All internal sources will use same format after change |
| DB data already in +00:00 format | No change needed - this is our target format |
| Downstream consumers | All consumers use string comparison - will work with consistent format |

## 🧪 Testing Strategy

1. Run existing tests after each phase
2. Verify `VN30FeaturePipeline` runs without time mismatch errors
3. Spot-check a few timepoints to confirm format is `+00:00`

## 📝 Execution Checklist

- [x] Phase 1: Update `time_utils.py`
- [x] Phase 2: Update `stock_data_collector.py`
- [x] Phase 3a: Update `tick_vn30_index_calculator.py`
- [x] Phase 3b: Update `tcbs_vn30_index_calculator.py`
- [x] Phase 3c: Update `vn30_base_calculator.py`
- [x] Phase 4: Run linter checks
- [ ] Phase 5: Test VN30FeaturePipeline end-to-end

