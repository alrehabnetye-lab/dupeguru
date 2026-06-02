# Duplicate File Detection Algorithm in dupeGuru

## Overview

dupeGuru is a cross-platform tool that identifies duplicate files using multiple intelligent scanning strategies. The algorithm implements a multi-stage approach that combines metadata comparison, content hashing, and fuzzy matching to accurately detect duplicates while maintaining high performance.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Scan Types](#scan-types)
3. [Core Matching Strategies](#core-matching-strategies)
4. [Algorithm Pipeline](#algorithm-pipeline)
5. [Implementation Details](#implementation-details)
6. [Performance Optimization](#performance-optimization)
7. [Group Formation Logic](#group-formation-logic)
8. [Reference File Selection](#reference-file-selection)

---

## Architecture Overview

The duplicate detection system is organized into three main components:

### 1. **Scanner Class** (`core/scanner.py`)
- Entry point for the scanning process
- Configures scan parameters and options
- Orchestrates the overall workflow
- Handles group formation and refinement

### 2. **Engine Module** (`core/engine.py`)
- Implements core matching algorithms
- Manages word extraction and comparison
- Builds word dictionaries and performs fuzzy matching
- Handles hash-based content comparison

### 3. **File System Module** (`core/fs.py`)
- Manages file metadata (size, hashes, paths)
- Caches hash values for performance
- Provides file existence checks

---

## Scan Types

dupeGuru supports multiple scanning methods, each optimized for different use cases:

| Scan Type | Purpose | Comparison Method | Exact Match |
|-----------|---------|-------------------|------------|
| `FILENAME` | Detect similar filenames | Fuzzy word matching | No |
| `FIELDS` | Match structured metadata (artist-title-album) | Field-based fuzzy matching | No |
| `FIELDSNOORDER` | Match fields regardless of order | Order-independent fuzzy matching | No |
| `TAG` | Compare music/media tags | Tag extraction + fuzzy matching | No |
| `FOLDERS` | Identify duplicate folder structures | Hierarchical path matching | No |
| `CONTENTS` | Find byte-identical files | Hash comparison | Yes |
| `FUZZYBLOCK` | Find similar images (Picture Edition) | Block-based image analysis | No |
| `EXIFTIMESTAMP` | Match images by EXIF date (Picture Edition) | Timestamp comparison | No |

---

## Core Matching Strategies

### A. Content-Based Matching (Exact Duplicates)

**Function:** `engine.getmatches_by_contents(files, bigsize=0, j=job.nulljob)`

#### Process Flow:

```
1. GROUP BY FILE SIZE
   └─ Only files with identical sizes can be duplicates
   
2. FOR EACH SIZE GROUP (with 2+ files):
   └─ FIRST PASS: Compare partial hash
      ├─ Hash first 4KB (configurable)
      ├─ Hash last 4KB (configurable)
      └─ If partial hashes differ → files are different
      
   └─ SECOND PASS (if bigsize > 0 and file > bigsize):
      ├─ Compute sample hash (blocks from middle)
      └─ If samples differ → files are different
      
   └─ THIRD PASS: Compare full content hash
      ├─ Compute complete SHA-1 hash
      └─ If hashes match → 100% match found
```

#### Optimization Strategy:

- **Tier 1 (Fast):** Size comparison - O(1)
- **Tier 2 (Medium):** Partial hash - O(log n) where n is file size
- **Tier 3 (Slow):** Full hash - O(n)
- **Special handling:** Skips comparing two reference files to save time

#### Example Thresholds:
```python
CHUNK_SIZE = 1024 * 1024  # 1 MiB
MIN_FILE_SIZE = 3 * CHUNK_SIZE  # 3 MiB (before using partial hash)
PARTIAL_OFFSET = (0x4000, 0x4000)  # 16KB from start and end
```

---

### B. Name/Metadata-Based Matching (Fuzzy Duplicates)

**Function:** `engine.getmatches(objects, min_match_percentage=80, ...)`

#### Step 1: Word Extraction

**Function:** `getwords(s)` and `getfields(s)`

```python
def getwords(s):
    """
    1. Normalize Unicode (NFD decomposition)
       "Café" → "Cafe`" (decomposed)
    
    2. Replace special characters with spaces
       "file_name-v2.txt" → "file name v2 txt"
    
    3. Filter to ASCII/digits/whitespace (with Unicode >894 support)
    
    4. Split on spaces and remove empty elements
       "file  name" → ["file", "name"]
    """
```

#### Step 2: Build Word Dictionary

**Function:** `build_word_dict(objects, j=job.nulljob)`

```
INPUT:  [File1(words=["photo", "album", "2024"]),
         File2(words=["photo", "album", "2023"]),
         File3(words=["image", "collection", "2024"])]

OUTPUT: {
    "photo": {File1, File2},
    "album": {File1, File2},
    "2024": {File1, File3},
    "2023": {File2},
    "image": {File3},
    "collection": {File3}
}
```

#### Step 3: Reduce Common Words

**Function:** `reduce_common_words(word_dict, threshold=50)`

**Goal:** Remove overly common words that appear in 50+ files (false positives)

**Strategy:**
```
For each common word:
  ├─ Check if any file has ONLY uncommon words
  ├─ If yes → keep the word (files would be unmatched otherwise)
  └─ If no → remove the word from dictionary
```

**Example:**
```
Word "the" appears in 200 files → normally discarded
But if File A has only word "the" → keep it (otherwise no matches)
```

#### Step 4: Merge Similar Words (Optional)

**Function:** `merge_similar_words(word_dict)`

**Purpose:** Group typos and spelling variations

**Algorithm:**
```
For each word in sorted dictionary (shortest first):
  ├─ Find similar words using difflib (0.8 similarity threshold)
  ├─ Merge all objects from similar words into the shortest word
  └─ Remove duplicate word entries
  
Example:
"photo" + "foto" + "phto" → merged under "foto"
(all files listed under shortest word)
```

#### Step 5: Compare Files Using Word Matches

**Function:** `compare(first, second, flags=())`

**Algorithm:**
```
INPUT: File1: ["movie", "action", "2024", "hd"]
       File2: ["movie", "action", "2023"]

PROCESS:
1. Create copy of second file's words (mutable)
   second_copy = ["movie", "action", "2023"]

2. For each word in first:
   ├─ "movie": found in second_copy → match! remove it
   ├─ "action": found in second_copy → match! remove it
   ├─ "2024": NOT found → no match
   ├─ "hd": NOT found → no match
   
3. Calculate percentage:
   matched_count = 2
   total_words = (4 + 3) = 7
   percentage = (2 * 2) / 7 * 100 = 57%

OUTPUT: 57% match
```

**Matching Flags:**

| Flag | Effect |
|------|--------|
| `WEIGHT_WORDS` | Longer words count more (3-letter word = 3x weight) |
| `MATCH_SIMILAR_WORDS` | Apply fuzzy matching within word comparison (typos match) |
| `NO_FIELD_ORDER` | Match fields in any order |

#### Step 6: Perform Comparative Matching

**Algorithm:**
```
MAIN LOOP - Process word dictionary:
  │
  ├─ Pop word from dictionary
  ├─ Get all files containing that word
  │  
  ├─ FOR EACH FILE in group:
  │  ├─ Compare against all OTHER files in group
  │  ├─ Skip already compared pairs (memoization)
  │  ├─ Calculate match percentage
  │  └─ If ≥ min_match_percentage → add to results
  │
  └─ Move to next word

Memory Optimization:
  - Pop items from dictionary as processed (free memory)
  - Track compared pairs to avoid duplicate comparisons
  - Handle MemoryError gracefully (return partial results)
```

**Complexity Analysis:**
- Time: O(W × F²) where W = unique words, F = files per word
- Space: O(W × F) for word dictionary
- Practical: Most files have < 50 words, keeping searches fast

---

## Algorithm Pipeline

### Complete Scanning Workflow

```
ENTRY POINT: Scanner.get_dupe_groups(files, ignore_list, progress_job)
│
├─ PHASE 1: Prepare Files
│  ├─ Add "is_ref" attribute to all files (default: False)
│  ├─ Remove duplicate paths (same path normalized)
│  ├─ Validate file existence and accessibility
│  │
│  └─ RESULT: Deduplicated file list
│
├─ PHASE 2: Match Files
│  │
│  ├─ IF scan_type in [CONTENTS, FOLDERS]:
│  │  └─ Call getmatches_by_contents()
│  │     └─ RESULT: List of Match(file1, file2, 100%)
│  │
│  └─ ELSE (FILENAME, FIELDS, TAG, etc.):
│     ├─ Extract words from each file
│     ├─ Build word dictionary
│     ├─ Reduce common words
│     ├─ Optionally merge similar words
│     ├─ Compare all files sharing words
│     │
│     └─ RESULT: List of Match(file1, file2, percentage)
│
├─ PHASE 3: Filter Matches
│  ├─ Remove folder parent-child duplicates
│  ├─ Filter by file extension (if mix_file_kind=False)
│  ├─ Check file existence (if include_exists_check=True)
│  ├─ Remove ref-to-ref matches (if not content scan)
│  ├─ Apply ignore_list rules
│  │
│  └─ RESULT: Filtered match list
│
├─ PHASE 4: Group Matches
│  ├─ Sort matches by percentage (highest first)
│  ├─ Build groups ensuring transitivity:
│  │  ├─ File A matches B (80%)
│  │  ├─ File B matches C (75%)
│  │  └─ Group: [A, B, C] (even if A-C only 60%)
│  ├─ Handle orphan matches recursively
│  │
│  └─ RESULT: List of Group objects
│
├─ PHASE 5: Select Reference Files
│  ├─ Prioritize by key function (_key_func)
│  ├─ Apply tie-breaker rules (_tie_breaker)
│  │
│  └─ RESULT: Groups with ref file at position 0
│
└─ PHASE 6: Return Results
   └─ Filter groups with at least one non-ref file
```

---

## Implementation Details

### Match Class

```python
class Match(namedtuple("Match", "first second percentage")):
    """
    Represents a matched pair of files.
    
    Attributes:
        first: File object (first file in pair)
        second: File object (second file in pair)
        percentage: int 0-100 indicating match level
                    (100 = exact content match, <100 = fuzzy match)
    """
```

### Group Class

The `Group` class intelligently aggregates match pairs:

```python
class Group:
    """Manages multiple matching files."""
    
    ref: File           # Reference file (not to be deleted)
    ordered: List[File] # All files sorted by priority
    unordered: Set[File] # Same files as unordered set
    dupes: List[File]  # ordered[1:] (files to potentially delete)
    matches: Set[Match] # All match pairs in group
    percentage: int    # Average match % with ref
    
    # Key Methods:
    add_match(match)        # Add match pair, validate consistency
    switch_ref(new_ref)     # Change which file is reference
    prioritize(key_func, tie_breaker)  # Reorder by criteria
    discard_matches()       # Free memory after grouping
```

#### Group Formation Algorithm

```python
def get_groups(matches):
    """Convert match pairs into coherent groups."""
    
    matches.sort(key=lambda m: -m.percentage)  # Highest % first
    
    dupe2group = {}  # Track which group each file belongs to
    groups = []
    
    for match in matches:
        first, second, _ = match
        
        # Check if files already in groups
        first_group = dupe2group.get(first)
        second_group = dupe2group.get(second)
        
        if first_group and second_group:
            # Both in groups
            if first_group is second_group:
                # Same group → add match
                target_group = first_group
            else:
                # Different groups → skip (would violate transitivity)
                continue
        elif first_group:
            # First in group, second not → add second to first's group
            target_group = first_group
            dupe2group[second] = target_group
        elif second_group:
            # Second in group, first not → add first to second's group
            target_group = second_group
            dupe2group[first] = target_group
        else:
            # Neither in group → create new group
            target_group = Group()
            groups.append(target_group)
            dupe2group[first] = target_group
            dupe2group[second] = target_group
        
        # Add match to target group
        target_group.add_match(match)
    
    # Handle orphan matches (found later) recursively
    return groups
```

---

## Performance Optimization

### 1. **Tiered Content Comparison**

```
File Size < 3 MiB:
  └─ Full hash comparison (fast enough)

File Size ≥ 3 MiB:
  ├─ Partial hash (16KB start + 16KB end)
  ├─ If different → files differ (skip full hash)
  └─ If same → sample hash → full hash (if needed)
```

### 2. **Word Dictionary Optimization**

```
Memory Management:
  ├─ Build dictionary incrementally
  ├─ Pop processed items (free memory)
  ├─ Cap results at 5,000,000 matches
  └─ Handle MemoryError gracefully
```

### 3. **Deduplication Mechanisms**

```
Compared Files Tracking:
  compared = {
    file1: {file2, file3, file5},  # Already compared with these
    file2: {file1, file4},
    ...
  }
  
Prevents O(n²) redundant comparisons
```

### 4. **Memoization**

- Cache extracted words per file
- Cache match pairs computed
- Cache group membership decisions

---

## Reference File Selection

### Tie-Breaking Rules

When multiple files have the same size, dupeGuru applies these rules in order:

```python
def _tie_breaker(ref, dupe):
    """
    Returns True if dupe should become reference instead of ref.
    Rules applied in order of priority:
    """
    
    refname = rem_file_ext(ref.name).lower()
    dupename = rem_file_ext(dupe.name).lower()
    
    # Rule 1: Prefer files WITHOUT "copy" in name
    if "copy" in dupename:
        return False  # dupe has "copy" → keep ref
    if "copy" in refname:
        return True   # ref has "copy" → swap to dupe
    
    # Rule 2: Prefer original over numbered duplicates
    # Example: "photo.jpg" preferred over "photo (1).jpg"
    if is_same_with_digit(dupename, refname):
        return False  # dupe is numbered version → keep ref
    if is_same_with_digit(refname, dupename):
        return True   # ref is numbered version → swap
    
    # Rule 3: Prefer files in shallower directories
    # (root folder preferred over nested folders)
    if len(dupe.path.parts) > len(ref.path.parts):
        return False  # dupe is deeper → keep ref
    
    return False  # Otherwise keep current ref
```

**Examples:**

| Reference | Duplicate | Selection | Reason |
|-----------|-----------|-----------|--------|
| `photo.jpg` | `photo (copy).jpg` | Keep reference | No "copy" in ref |
| `song (1).mp3` | `song.mp3` | Duplicate becomes ref | Dupe is original |
| `/photo.jpg` | `/archive/photo.jpg` | Keep reference | Ref is shallower |
| `document.pdf` (100KB) | `document.pdf` (100KB) | Keep reference | Same precedence |

---

## Configurable Parameters

### Scanner Configuration

```python
class Scanner:
    # Content scanning
    big_file_size_threshold = 0  # Trigger sample hash for files > this
    size_threshold = 0           # Minimum file size to include
    large_size_threshold = 0     # Maximum file size to include
    
    # Name/metadata scanning
    min_match_percentage = 80    # Minimum % match to report
    match_similar_words = False  # Enable fuzzy word matching
    word_weighting = False       # Weight by word length
    scanned_tags = {"artist", "title"}  # For music files
    
    # Group filtering
    mix_file_kind = True         # Allow different extensions in group
    include_exists_check = True  # Verify files still exist
    scan_type = ScanType.FILENAME
```

---

## Example: Scanning by Filename

### Input Files:
```
Files: [
    File(name="vacation_photo_2024.jpg", size=2500000),
    File(name="vacation_photo_2024_edited.jpg", size=2500000),
    File(name="beach_photo_2024.jpg", size=3000000),
]
```

### Step-by-Step Execution:

#### Phase 1: Extract Words
```
File 1: ["vacation", "photo", "2024"]
File 2: ["vacation", "photo", "2024", "edited"]
File 3: ["beach", "photo", "2024"]
```

#### Phase 2: Build Dictionary
```
Dictionary: {
    "vacation": {File1, File2},
    "photo": {File1, File2, File3},
    "2024": {File1, File2, File3},
    "edited": {File2},
    "beach": {File3},
}
```

#### Phase 3: Compare Files

**Comparison 1: File 1 vs File 2 (via word "vacation")**
```
File1 words: ["vacation", "photo", "2024"]
File2 words: ["vacation", "photo", "2024", "edited"]

Matches: "vacation", "photo", "2024" (3 words)
Total: (3 + 4) = 7 words
Percentage: (3 × 2) / 7 × 100 = 85.7% ≈ 86%
Result: Match(File1, File2, 86)  ✓ Above 80% threshold
```

**Comparison 2: File 1 vs File 3 (via word "photo")**
```
File1 words: ["vacation", "photo", "2024"]
File3 words: ["beach", "photo", "2024"]

Matches: "photo", "2024" (2 words)
Total: (3 + 3) = 6 words
Percentage: (2 × 2) / 6 × 100 = 66.7% ≈ 67%
Result: Match(File1, File3, 67)  ✗ Below 80% threshold
```

**Comparison 3: File 2 vs File 3 (via word "photo")**
```
File2 words: ["vacation", "photo", "2024", "edited"]
File3 words: ["beach", "photo", "2024"]

Matches: "photo", "2024" (2 words)
Total: (4 + 3) = 7 words
Percentage: (2 × 2) / 7 × 100 = 57.1% ≈ 57%
Result: Match(File2, File3, 57)  ✗ Below 80% threshold
```

#### Phase 4: Group Formation
```
Matches >= 80%: [Match(File1, File2, 86)]

Groups formed: 1 group containing [File1, File2]
               Reference: File1 (or File2 depending on tie-breaker)
```

#### Phase 5: Output
```
Group 1:
  - Reference: vacation_photo_2024.jpg (2.5 MB)
  - Duplicate: vacation_photo_2024_edited.jpg (2.5 MB) [86% match]
```

---

## Algorithm Complexity Analysis

### Time Complexity

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Word extraction | O(N × M) | N files, M avg words per file |
| Build dictionary | O(N × M) | Linear scan of files and words |
| Reduce common | O(W) | W unique words |
| Merge similar | O(W² × log W) | Quadratic similarity checks |
| Compare files | O(W × F²) | W words, F files per word |
| Group formation | O(M) | M match pairs |
| **Total** | **O(W × F²)** | Dominated by file comparison |

### Space Complexity

| Data Structure | Space | Notes |
|---|---|---|
| Word dictionary | O(W × F) | W words, F files per word |
| Compared pairs | O(F²) | Track all file pairs |
| Groups | O(N) | N files in groups |
| **Total** | **O(W × F)** | Dictionary dominates |

### Best/Worst Case Scenarios

**Best Case:** Identical files (different sizes)
- Time: O(1) per pair (size comparison)
- Example: 10,000 files → 1 second

**Average Case:** Similar filenames
- Time: O(M × log M) where M = matches found
- Example: 10,000 files → 5-30 seconds

**Worst Case:** Highly similar files
- Time: O(N²) comparisons
- Example: 1,000 files of same size → 1-5 minutes
- Mitigation: Reduce common words, filter by threshold

---

## Integration with UI

The scanner is designed for async operation:

```python
# Usage example:
scanner = Scanner()
scanner.scan_type = ScanType.FILENAME
scanner.min_match_percentage = 80

groups = scanner.get_dupe_groups(
    files=file_list,
    ignore_list=ignore_patterns,
    j=job_progress_callback  # Updates UI during scan
)

for group in groups:
    print(f"Reference: {group.ref.name}")
    for dupe in group.dupes:
        print(f"  Duplicate: {dupe.name} ({group.percentage}%)")
```

---

## Summary

dupeGuru's duplicate detection algorithm is a **sophisticated multi-stage system** that:

1. ✅ **Supports multiple matching strategies** (content, name, metadata)
2. ✅ **Optimizes for performance** (tiered hashing, word filtering)
3. ✅ **Handles edge cases** (ref files, mixed types, path duplicates)
4. ✅ **Provides intelligent grouping** (transitive matching)
5. ✅ **Selects smart references** (preserves originals over copies)
6. ✅ **Gracefully handles errors** (memory limits, missing files)

The algorithm is production-ready and has been battle-tested across millions of files on diverse systems (Linux, macOS, Windows).

---

## References

- **Scanner Implementation:** `core/scanner.py`
- **Engine/Matching Logic:** `core/engine.py`
- **File System Utilities:** `core/fs.py`
- **Main Entry Point:** `core/app.py`

For more information, visit: https://dupeguru.voltaicideas.net/
