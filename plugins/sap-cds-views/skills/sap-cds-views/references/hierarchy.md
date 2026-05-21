# CDS Hierarchy Reference

## When to Use CDS Hierarchy

```
Use CDS hierarchy when:
  - Data has parent-child relationships (storage bins → sections → zones → warehouse)
  - Need tree display in Fiori TreeTable / treeGrid
  - Require level-based aggregation (total weight by zone/section/bin)
  - Need cycle detection in recursive structures

SAP standard examples:
  - Organizational hierarchy (company → plant → storage location)
  - Material BOM (header → assembly → component)
  - Storage structure (warehouse → storage type → section → bin)
  - Cost center hierarchy
```

## CDS Hierarchy Definition

```abap
" Step 1: Base CDS view with the node data
define view entity ZI_StorageStructure
  as select from /scwm/lgpla as pla
  left outer join /scwm/lgber as ber
    on pla.lgnum = ber.lgnum and pla.lgber = ber.lgber
{
  key pla.lgnum    as Lgnum,
  key pla.lgpla    as StorageBin,
      pla.lgber    as Section,
      pla.lgtyp    as StorageType,
      pla.parent_pla as ParentBin,     " FK to parent storage bin
      pla.bin_type   as BinType,
      ber.section_name as SectionName,

      @Semantics.quantity.unitOfMeasure: 'WeightUOM'
      pla.max_weight as MaxWeight,
      'KG'           as WeightUOM
}

" Step 2: Define the hierarchy
define hierarchy ZH_StorageStructure
  as parent child hierarchy(
    source ZI_StorageStructure
    child to parent association _ParentBin
    start where ParentBin is initial    " roots: bins with no parent
    siblings order by StorageBin ascending
  )
{
  Lgnum,
  StorageBin,
  Section,
  StorageType,
  ParentBin,
  BinType,
  SectionName,
  MaxWeight,
  WeightUOM,

  " Hierarchy meta-fields (added automatically by hierarchy engine):
  $node.node_id             as NodeId,
  $node.parent_id           as ParentId,
  $node.hierarchy_level     as HierarchyLevel,
  $node.hierarchy_rank      as HierarchyRank,
  $node.hierarchy_tree_size as SubtreeSize,
  $node.is_leaf_node        as IsLeafNode       " X = leaf (no children)
}
```

## Hierarchy with Back-Association

```abap
" The hierarchy source view needs a to-parent association:
define view entity ZI_StorageStructure
  as select from /scwm/lgpla as pla
  association [0..1] to ZI_StorageStructure as _ParentBin
    on $projection.ParentBin = _ParentBin.StorageBin
   and $projection.Lgnum     = _ParentBin.Lgnum
{
  key pla.lgnum     as Lgnum,
  key pla.lgpla     as StorageBin,
      pla.parent_pla as ParentBin,
      pla.lgtyp      as StorageType,
      pla.lgber      as Section,
      _ParentBin
}

" In the hierarchy definition:
define hierarchy ZH_StorageStructure
  as parent child hierarchy(
    source ZI_StorageStructure
    child to parent association _ParentBin
    ...
  )
```

## Hierarchy Parameters

```abap
define hierarchy ZH_OrgHierarchy
  as parent child hierarchy(
    source ZI_OrgNode
    child to parent association _Parent

    " Start (root) condition:
    start where ParentId is initial         " nulls = roots
    " OR: start where NodeType = 'ROOT'

    " Siblings ordering:
    siblings order by NodeCode ascending

    " Cycle handling (prevent infinite loops):
    with orphan nodes                        " include nodes whose parent is missing
    with cycle detection                     " error on cycles

    " Depth limit:
    " (no keyword — use WHERE $node.hierarchy_level <= 5 in SELECT)
  )
{
  NodeId,
  ParentId,
  NodeName,
  NodeType,
  $node.hierarchy_level  as Level,
  $node.is_leaf_node     as IsLeaf
}
```

## Selecting from a Hierarchy

```abap
" SELECT from hierarchy in ABAP (ABAP 7.54+):
SELECT lgnum,
       storage_bin,
       section,
       parent_bin,
       hierarchy_level,
       is_leaf_node,
       subtree_size
  FROM HIERARCHY_DESCENDANTS(
    SOURCE zh_storage_structure
    START WHERE lgnum = '1710' AND parent_bin = ''   " roots of warehouse 1710
    DEPTH 3                                          " max 3 levels deep
  )
  INTO TABLE @DATA(lt_hierarchy).

" All descendants of a specific node:
SELECT * FROM HIERARCHY_DESCENDANTS(
  SOURCE zh_storage_structure
  START WHERE storage_bin = 'A-01'   " subtree under bin A-01
)
INTO TABLE @DATA(lt_subtree).

" Ancestors (path to root):
SELECT * FROM HIERARCHY_ANCESTORS(
  SOURCE zh_storage_structure
  START WHERE storage_bin = 'A-01-01-001'
)
INTO TABLE @DATA(lt_ancestors).
```

## Fiori TreeTable Integration

```json
// manifest.json — Object Page with tree section
"targets": {
  "StorageStructurePage": {
    "type": "Component",
    "name": "sap.fe.templates.ObjectPage",
    "options": {
      "settings": {
        "entitySet": "StorageStructure",
        "navigation": {
          "StorageStructure": {
            "detail": { "route": "StorageStructurePage" }
          }
        }
      }
    }
  }
}
```

```abap
" CDS projection for TreeTable display:
@UI.lineItem: [
  { position: 10, label: 'Bin' },
  { position: 20, label: 'Section' },
  { position: 30, label: 'Type' },
  { position: 40, label: 'Level' },
  { position: 50, label: 'Max Weight' }
]

define view entity ZC_StorageStructure
  as projection on ZI_StorageStructure
{
  key Lgnum,
  key StorageBin,
      Section,
      StorageType,
      ParentBin,
      MaxWeight,
      WeightUOM,
      _ParentBin : redirected to parent ZC_StorageStructure
}
```

## EWM Storage Structure Hierarchy

```abap
" Real-world: EWM warehouse → storage type → section → bin
define view entity ZI_EwmStorageNode
  as select from /scwm/lgpla as bin
  left outer join /scwm/t331 as stype
    on bin.lgnum = stype.lgnum and bin.lgtyp = stype.lgtyp
  association [0..1] to ZI_EwmStorageNode as _Parent
    on bin.lgnum     = _Parent.Lgnum
   and bin.parent_pla = _Parent.StorageBin
{
  key bin.lgnum      as Lgnum,
  key bin.lgpla      as StorageBin,
      bin.lgber      as Section,
      bin.lgtyp      as StorageType,
      stype.ltxt     as StorageTypeDesc,
      bin.parent_pla as ParentBin,
      bin.lptyp      as BinType,

      @Semantics.quantity.unitOfMeasure: 'WeightUOM'
      bin.maxgew     as MaxWeight,
      bin.meins      as WeightUOM,

      _Parent
}

define hierarchy ZH_EwmStorageStructure
  as parent child hierarchy(
    source ZI_EwmStorageNode
    child to parent association _Parent
    start where ParentBin is initial
    siblings order by StorageBin ascending
    with orphan nodes
  )
{
  Lgnum,
  StorageBin,
  StorageType,
  StorageTypeDesc,
  Section,
  ParentBin,
  BinType,
  MaxWeight,
  WeightUOM,
  $node.hierarchy_level  as Level,
  $node.is_leaf_node     as IsLeaf,
  $node.hierarchy_rank   as Rank
}
```
