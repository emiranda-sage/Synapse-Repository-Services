# PLFM-9997 — Expand Dataset and DatasetCollection rows in query-mode Download List adds

*Working notes for the code review. Not committed.*

## The bug

An EntityView's `viewTypeMask` is a bitmask, so one view can index `File`, `Dataset` and
`DatasetCollection` entities together. Adding such a query result to a download list silently dropped
every non-file row: `DownloadListManagerImpl.addQueryResultsToDownloadList` passed the page through
`DownloadListDAO.filterUnsupportedTypes`, whose SQL matches only `NODE_TYPE IN ('file','recordset')`.
No error, no indication anything was lost.

The `add/stats` preview endpoint had the mirror problem — it counted each Dataset row as one file — so
the preview and the add already disagreed before this change.

> The ticket guesses the rows drop because a Dataset has no file handle. It doesn't: file handles are
> never consulted on the add path (only `(principal, entity id, version)` is written; handles resolve
> at read time). The rows die one step earlier, at a deliberate entity-type filter.

## What changed

| File | Change |
|---|---|
| `DownloadListDAO` / `Impl` | New `groupItemsByType` — one `NODE` query answers "which rows are files, which are Datasets, which are DatasetCollections". |
| `DownloadListDAO` / `Impl` | New `expandDatasetCollectionRefs` — flattens DatasetCollection refs into the Dataset refs their items point to, via `JSON_TABLE`. |
| `DownloadListDAO` / `Impl` | New `countFileRefsInDatasets` — counts the files that are both a given result row and a member of a given Dataset, so the stats count does not report a file twice. |
| `DownloadListManagerImpl` | Add path: split each page by type; files insert directly, Dataset/DatasetCollection refs merge into one de-duplicated set feeding the *existing* expansion call. |
| `DownloadListManagerImpl` | Stats path: count derived from the row scan, minus the file/Dataset overlap; size topped up for collection-reached files. |
| `AddToDownloadListRequest.json`, `AddToDownloadListStatsResponse.json` | Document the new behavior and the remaining imprecision. |

`filterUnsupportedTypes` itself is **not** modified. The query path no longer calls it — that call is
replaced by `groupItemsByType` — but the method is untouched and still serves the explicit-batch add path
(`addBatchOfFiles`), which only ever needs the yes/no answer.

**No new insertion SQL.** Both paths reuse `addDatasetEntityRefFilesToDownloadList` and
`getAddDatasetEntityRefFilesToDownloadListStats`, which already expanded a dataset's `items` for the
`parentId` mode. The two new queries are both read-only: `expandDatasetCollectionRefs` and
`countFileRefsInDatasets`.

## The changes, in code

### 1. `DownloadListDAOImpl.groupItemsByType` — classify each row by entity type

The existing `filterUnsupportedTypes` answers a yes/no question — "is this a file?" — and discards
everything that answers no. The query path needs three answers instead, because each type is handled
differently: files insert directly, Datasets expand into their member files, DatasetCollections expand
one level further. `groupItemsByType` answers "what type is each of these?" in one `NODE` query.

```java
namedJdbcTemplate.query("SELECT ID, NODE_TYPE FROM NODE WHERE ID IN (:ids) AND NODE_TYPE IN (:types)",
        params, (RowCallbackHandler) rs -> idToType.put(rs.getLong(COL_NODE_ID),
                EntityType.valueOf(rs.getString(COL_NODE_TYPE))));

// Iterating the batch (not the ResultSet) is what preserves the caller's row order.
for (DownloadListItem item : batch) {
    EntityType type = idToType.get(KeyFactory.stringToKey(item.getFileEntityId()));
    if (type != null) {
        itemsByType.computeIfAbsent(type, t -> new ArrayList<>()).add(item);
    }
}
```

The two methods each run their own query against `NODE`. An earlier revision of this change had
`filterUnsupportedTypes` delegate to `groupItemsByType`, on the grounds that one query in the class beats
two. That was reverted: flattening a type-grouped map reorders items across types (a batch of
`[file, recordset, file]` comes back as `[file, file, recordset]`), so the caller had to use the map only
as an id lookup and then re-scan the original list to preserve order — four steps, two explanatory
comments, and two new regression tests, to replace a three-line query. The duplication is ~4 lines of
similar SQL, which the project's "write SQL inline where it's used" guidance prefers over the
indirection.

### 2. `DownloadListDAOImpl.expandDatasetCollectionRefs`

Same `JSON_TABLE` technique the file already uses to unpack a Dataset's `items` into files, applied one
level shallower to unpack a DatasetCollection's `items` into dataset refs — for any number of
collections in one round trip.

```sql
SELECT DISTINCT D.OWNER_NODE_ID AS id, D.NUMBER AS version
FROM NODE_REVISION AS C
JOIN JSON_TABLE(C.ITEMS, '$[*]' COLUMNS (
        id VARCHAR(30) PATH '$.entityId',
        version BIGINT PATH '$.versionNumber')) AS D_REF
-- the join drops items that do not resolve to a real revision, instead of letting a null or
-- non-numeric entityId/versionNumber become entity 0 / version 0
JOIN NODE_REVISION AS D ON (
        D.OWNER_NODE_ID = CAST(REPLACE(D_REF.id, 'syn', '') AS UNSIGNED)
        AND D.NUMBER = D_REF.version)
WHERE (C.OWNER_NODE_ID, C.NUMBER) IN (:collections)
```

`WHERE (OWNER_NODE_ID, NUMBER) IN (...)` is the load-bearing part: it pins the expansion to the
*version the query row carried*, and resolves every collection in the batch in one round trip
(tradeoff 3).

### 3. `addQueryResultsToDownloadList` — the add path, split by type

```java
do {
    Pair<Long, Map<EntityType, List<DownloadListItem>>> page = queryItemsByType(...limit, offset);
    Map<EntityType, List<DownloadListItem>> itemsByType = page.getSecond();

    totalFilesAdded += downloadListDao.addBatchOfFilesToDownloadList(
            userInfo.getId(), selectSupportedFileItems(itemsByType));

    // Hoisted above the expansion: it derives a LIMIT from the remaining capacity, and MySQL
    // rejects a negative LIMIT.
    checkQueryCapacity(totalFilesAdded, usersDownloadListCapacity);

    List<DownloadListItem> datasetItems = itemsByType.getOrDefault(EntityType.dataset, List.of());
    List<DownloadListItem> collectionItems = itemsByType.getOrDefault(EntityType.datasetcollection, List.of());

    // ONE deduplicated set: a dataset reachable both directly and via a collection is expanded once.
    Set<EntityRef> datasetRefs = new LinkedHashSet<>(toEntityRefs(datasetItems, useVersion));
    if (!collectionItems.isEmpty()) {
        datasetRefs.addAll(downloadListDao.expandDatasetCollectionRefs(toEntityRefs(collectionItems, useVersion)));
    }

    if (!datasetRefs.isEmpty()) {
        long remainingCapacity = usersDownloadListCapacity - totalFilesAdded + 1;
        totalFilesAdded += downloadListDao.addDatasetEntityRefFilesToDownloadList(
                userInfo.getId(), new ArrayList<>(datasetRefs), remainingCapacity);
        checkQueryCapacity(totalFilesAdded, usersDownloadListCapacity);
    }

    offset += limit;
} while (pageSize >= limit);
```

Everything else in this method is unchanged — the SQL rewrite (now extracted), `cloneQuery`, the loop
condition, the exception mapping. `pageSize` is the **raw** row count taken before classification, so
loop termination is unaffected by non-file rows.

### 4. `getAddToDownLoadListStatsFromQuery` — the stats path

The count is built from the scan, not from the index (tradeoff 1):

```java
// The index is asked ONLY for the size sum now; runCount is gone.
QueryResultBundle indexResult = tableQueryManager.querySinglePage(progressCallback, userInfo, query,
        new QueryOptions().withRunSumFileSizes(true));

rewriteQueryToSelectFileColumn(query, useVersion);   // the same rewrite the add path performs

long fileRowCount = 0L;
Set<EntityRef> fileRefs = new LinkedHashSet<>();        // capped; see below
Set<EntityRef> datasetRefs = new LinkedHashSet<>();
Set<EntityRef> collectionRefs = new LinkedHashSet<>();
do {
    ...
    List<DownloadListItem> fileItems = selectSupportedFileItems(itemsByType);
    fileRowCount += fileItems.size();

    if (useVersion && !isFileRefSetTruncated) {         // retained for the overlap check (tradeoff 7)
        for (EntityRef fileRef : toEntityRefs(fileItems, useVersion)) { ... }
    }

    datasetRefs.addAll(toEntityRefs(itemsByType.getOrDefault(EntityType.dataset, List.of()), useVersion));
    collectionRefs.addAll(toEntityRefs(itemsByType.getOrDefault(EntityType.datasetcollection, List.of()), useVersion));
    pageOffset += MAX_QUERY_PAGE_SIZE;
} while (pageSize >= MAX_QUERY_PAGE_SIZE);
```

Everything retained across the loop is either a counter or a bounded set — the container refs are bounded
by the number of containers, and `fileRefs` by an explicit cap — so memory does not grow with the result
size. Then the size correction (tradeoff 2) and the overlap subtraction (tradeoff 7):

```java
Set<EntityRef> collectionOnlyRefs = new LinkedHashSet<>();
if (!collectionRefs.isEmpty()) {
    collectionOnlyRefs.addAll(downloadListDao.expandDatasetCollectionRefs(new ArrayList<>(collectionRefs)));
    collectionOnlyRefs.removeAll(datasetRefs);   // a direct row already contributed its size
}

Set<EntityRef> allRefs = new LinkedHashSet<>(datasetRefs);
allRefs.addAll(collectionOnlyRefs);

// Held in a local so the overlap below is measured against exactly the datasets whose files were counted.
List<EntityRef> cappedRefs = capContainerRefs(allRefs);
AddToDownloadListStatsResponse expandedStats = downloadListDao
        .getAddDatasetEntityRefFilesToDownloadListStats(cappedRefs);

// Bytes the index sum is missing, because a DatasetCollection carries no replicated aggregate size.
long collectionOnlyFileSize = collectionOnlyRefs.isEmpty() ? 0L : downloadListDao
        .getAddDatasetEntityRefFilesToDownloadListStats(capContainerRefs(collectionOnlyRefs)).getFileSize();

// A file can be both its own row and a member of one of those datasets. The add path stores it
// once, so counting it on both sides would promise more files than the add delivers.
long overlapCount = downloadListDao.countFileRefsInDatasets(new ArrayList<>(fileRefs), cappedRefs);

return new AddToDownloadListStatsResponse()
    .setFileCount(fileRowCount + expandedStats.getFileCount() - overlapCount)
    .setFileSize(indexFileSize + collectionOnlyFileSize)
    .setIsFileCountAndSizeEstimate(isFileSizeSampled || isRefCountTruncated || isFileRefSetTruncated);
```

`fileRefs` is collected during the same scan and capped at `FILE_STATS_MAX_FILE_REFS_COUNT` (10,000),
because unlike the container refs it grows with the number of rows rather than the number of containers.
It is only collected when `useVersion` is true: without a version a file row is stored against
`NULL_VERSION_NUMBER`, which can never equal the concrete version recorded in a dataset's `items`, so
the add path really does insert both and there is no overlap to remove.

Compare with what this replaced — the version that could go negative:

```java
.setFileCount(indexBasedStats.getFileCount()      // from the ORIGINAL query's COUNT(*)
    - datasetItems.size() - collectionItems.size() // rows found by scanning the REWRITTEN query
    + datasetStats.getFileCount())
```

## How it fits together

```
rewriteQueryToSelectFileColumn(query)        <- shared by both paths
  └─ per page: queryItemsByType(...)         <- shared by both paths
       ├─ file / recordset rows  ──────────► inserted (add) / counted (stats)
       └─ dataset + datasetcollection rows
              └─ expandDatasetCollectionRefs ─► merged into ONE deduplicated Set<EntityRef>
                     └─ addDatasetEntityRefFilesToDownloadList (add)
                        getAddDatasetEntityRefFilesToDownloadListStats (stats)
                          └─ countFileRefsInDatasets (stats only) ─► subtract the file/Dataset overlap,
                             which INSERT IGNORE already handles for the add path
```

`queryItemsByType` is the point of the design: both paths resolve rows through the identical rewrite
and the identical classification, so they cannot drift apart. That is requirement #2 of the ticket.

Merging into a single de-duplicated set *before* counting is what stops a Dataset that is both its own
row and a member of a DatasetCollection in the same result from being counted twice.

## Tradeoffs, and how each was decided

**1. `fileCount` comes from the scan, not from the index `COUNT(*)`.**
The first implementation subtracted the Dataset/DatasetCollection row count from the index's
`COUNT(*)` and added the expanded file count back. That is only valid if the count and the scan
enumerate the same rows, and they don't: the rewrite passes `null` as the `SetQuantifier`, so
**`DISTINCT` is dropped**, and `queryItemsByType` overwrites any request-level `limit`/`offset` with
its own paging values while the count honours them. Either one made `fileCount` go negative. Since
the scan already visits every row, the exact file-row count is free — so the subtraction is gone
entirely, and with it a whole class of bug.

**2. `fileSize` is left to the index for Dataset rows, and topped up for collection-only refs.**
A `Dataset`'s replication row already carries the aggregate size of its member files
(`DatasetMetadataProvider.validateEntity` → `nodeDao.getFileSummary`, surfaced by
`NodeDAOImpl.getEntityDTOs`), so adding the expansion's size on top would double count.
`DatasetCollectionMetadataProvider` has **no** equivalent, so bytes reachable only through a
collection are absent from the index sum and are added back explicitly — excluding any dataset that
was also a direct row, which already contributed. *Residual, documented in the schema:* a file that is
both its own result row and a dataset member still contributes its **size** twice. Its effect on
`fileCount` is removed by `countFileRefsInDatasets` (tradeoff 7); the size is left because
`SumFileSizesQuery` builds its row query with `LIMIT MAX_ROWS_PER_CALL + 1` and `MAX_ROWS_PER_CALL = 100`,
so the index sum is a 101-row sample for any result over 100 rows — above that threshold
`isFileCountAndSizeEstimate` is already set and exactness is not on offer.

`getAddDatasetEntityRefFilesToDownloadListStats` is therefore called twice when a collection is present:
once over every ref for the count, once over the collection-only refs for the size. An earlier revision
skipped the second call when there were no direct Dataset rows (the two sets are identical then), but the
nested conditional cost more in clarity than the round trip saved on a path that already issues one index
query per 10,000 rows.

**3. Batched `JSON_TABLE` SQL rather than looping `NodeDAO.getNodeItems`.**
Three reasons, strongest first:

- **Round trips.** `getNodeItems` is one query per node. A page holds up to `MAX_QUERY_PAGE_SIZE`
  (10,000) rows, every one of which could be a collection, and the stats path pages the *entire*
  result — so the loop is O(collections) queries against this one. The `parentId` path loops
  `getNodeItems` because it has exactly one collection; that does not generalise to a query result.
- **Stale rows.** `selectRevisionColumnValue` turns a missing revision into `NotFoundException`, so a
  loop needs per-item exception handling or one view-index row whose revision has since been deleted
  fails the whole add job. `WHERE (OWNER_NODE_ID, NUMBER) IN (...)` simply returns nothing for it.
- **Dangling member refs.** The `JOIN NODE_REVISION AS D` drops items pointing at a version that no
  longer exists, which nothing upstream prevents: `DatasetCollectionMetadataProvider.validateEntity`
  checks items through `getEntityHeader`, which is keyed on entity id and discards `versionNumber`.
  Weakest of the three — `addDatasetEntityRefFilesToDownloadList` and `...Stats` apply the same join
  downstream anyway, so this only matters for the 500-ref stats cap and the `removeAll` bookkeeping.

Note what is *not* a reason: version-awareness is not exclusive to SQL. A version-aware
`getNodeItems(IdAndVersion)` overload would be a few lines — `selectRevisionColumnValue` already
implements the version-pinned branch, and `getDefiningSql(IdAndVersion)` is precedent for the shape.
Batching and stale-row tolerance are what rule out the loop, not the ability to pin a version.

One level of expansion is always enough because `DatasetCollectionMetadataProvider.validateEntity`
rejects non-dataset items: *"Only dataset entities can be included in a dataset collection."*

**4. The stats path pages the whole result instead of sampling one page.**
Sampling 501 rows would miss a Dataset sitting further down a large view, making the preview disagree
with the add — the exact thing the ticket forbids. **Cost:** a file-only view now pays one extra index
query per 10,000 rows to discover it has no datasets. A `ViewTypeMask` guard (skip the scan when the
view cannot contain `Dataset`/`DatasetCollection`) would remove that cost and is the obvious follow-up;
it was left out here because it needs the view's scope type and only helps `entityview` targets.

**5. No provenance tracking ("the lazy way").**
Files added via a Dataset land in the flat list like any other file; there is no record of which
dataset they came from. Tracking it needs a new column (today's PK is
`(PRINCIPAL_ID, ENTITY_ID, VERSION_NUMBER)` with nowhere to put a source), a new response field, and
frontend work. Agreed out of scope with a senior teammate — this is the decision that keeps the change
this small.

**6. Dataset-contributed files ignore `useVersionNumber`.**
They are always pinned to the version recorded in the dataset's `items`. Not a new rule — it is what
the `parentId`-on-a-dataset path already does. It was simply undocumented, and now is.

**7. `fileCount` subtracts the overlap between file rows and dataset members.**
A file can be both its own result row and a member of a Dataset in that same result — a `File|Dataset`
view over a project holding both the files and a Dataset built from them. The add path inserts it once
(the list is keyed on `(PRINCIPAL_ID, ENTITY_ID, VERSION_NUMBER)` and both inserts use `INSERT IGNORE`,
which returns rows *inserted*), so counting it on both sides would promise more files than the add
delivers — breaking requirement #2. This is why the **add path needs no equivalent fix**: the database
already de-duplicates for it.

The intersection is computed in SQL by `countFileRefsInDatasets`, not in Java, because the file rows come
from the **index** database and the dataset members from the **main** database — no single join can see
both, so one side has to be materialized. The query is keyed on the bounded side (the members of at most
500 datasets) and takes the file refs as a parameter, capped at `FILE_STATS_MAX_FILE_REFS_COUNT` (10,000)
since that set grows with row count rather than container count.

Only applies when `useVersion` is true. Without a version a file row is stored against
`NULL_VERSION_NUMBER` (-1), which can never equal the concrete version in a dataset's `items`, so the add
path genuinely inserts both and there is nothing to subtract. The DAO short-circuits on the empty ref
list, so that case costs no query.

Not a regression this change introduced — the old `COUNT(*)` disagreed here too, reporting
`N_files + 1` for the Dataset row. It is the last gap between the preview and the add.

## How it's tested

| Layer | Count | What it actually proves |
|---|---|---|
| `DownloadListDaoImplTest` (real MySQL) | 121 pass | The SQL. Notably `testExpandDatasetCollectionRefsWithOldVersion` asserts that expanding a **version-1** collection ref returns version 1's datasets and a v2 ref returns v2's — the scenario the `getNodeItems` loop would get wrong. `...WithUnresolvableItem` proves items that don't resolve to a real revision are dropped. `testCountFileRefsInDatasetsWithFileInMultipleDatasets` proves the overlap is counted once, and `...WithDifferentVersion` that it is keyed on `(id, version)` rather than on id. |
| `DownloadListManagerImplTest` (mocked) | 131 pass | The branching and arithmetic: mixed pages, collection expansion, the dedup guard on both paths, `useVersionNumber=false` on both paths, the 500-dataset cap, capacity breaches, the overlap subtraction, and the exact rewritten SQL and page offsets on both paths. |
| `TableViewIntegrationTest` (real view index) | 2 new, both pass | `testAddViewQueryToDownloadListWithDatasetRows`: a `File\|Dataset` view over 3 files plus a Dataset whose 2 members sit **outside** the view scope — asserts `stats.getFileCount() == addResponse.getNumberOfFilesAdded()` (both 5). `testAddViewQueryToDownloadListWithFileAlsoInDataset`: the same view, but the Dataset's 2 members **are** view rows — asserts the two agree again (both 3), which is the overlap case that tradeoff 7 fixes. **This is requirement #2.** |

All three suites were run green against a real dev stack. Two fixes were verified by temporarily
reverting them and confirming the new test fails: the capacity check ordering, and the revision join in
`expandDatasetCollectionRefs`.

Diff size: **~1,950 lines, of which ~1,390 are tests**. The production delta is ~560 lines, a share of
which is the query rewrite moved out of inline code into `rewriteQueryToSelectFileColumn` so both paths
can share it.

## Known limits

- `fileSize` still double counts a file that is both its own result row and a dataset member (documented
  in `isFileCountAndSizeEstimate`). Pre-existing for the Dataset case, and moot above 100 rows where the
  index sum is already a sample. `fileCount` no longer does — see tradeoff 7.
- More than 500 distinct datasets in one result → only the first 500 are expanded;
  `isFileCountAndSizeEstimate` is set.
- More than `FILE_STATS_MAX_FILE_REFS_COUNT` (10,000) file rows alongside a Dataset → only the first
  10,000 are checked for the overlap above, so `fileCount` can still over-report;
  `isFileCountAndSizeEstimate` is set.
- The 500-ref cap is applied separately to `allRefs` and to `collectionOnlyRefs`, so with more than 500
  refs the reported count and size can describe different subsets. Capping once and deriving both from
  the capped set would fix it. Flagged as an estimate either way.
- `stats.getFileCount()` and `addResponse.getNumberOfFilesAdded()` agree only for a download list that
  does not already hold some of these files. The add path counts rows *inserted*, and `INSERT IGNORE`
  returns 0 for a file already on the list, so a user re-adding an overlapping query sees a smaller add
  figure than the preview promised. The preview is a property of the query; the add figure is a property
  of the query *and* the current list. The integration tests assert equality against a fresh list.
- The `+1` sentinel on the add path's remaining capacity is a best-effort over-capacity signal, not a
  guarantee: the DAO uses `INSERT IGNORE` and returns rows *inserted*, so files already on the user's
  list absorb the sentinel. A real fix needs a count-before-insert.
- The stats preview cost noted in tradeoff 4.
- `addDescendantsToDownloadList` still walks only `project`/`folder`, so a Dataset nested in a folder
  is still not expanded by `parentId` + `recursive`. Same class of bug, different path, not this ticket.
