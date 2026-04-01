# Data Set Creator And Generator Reference

## Purpose

`data_set_creator_and_generator.py` provides a configurable dataset-building pipeline with:

- question definitions
- dependencies and conditional mappings
- copied values
- derived logic fields
- grouping and pairwise coverage
- CSV, Excel, UI-payload, profile, and ACM-aware exports

The module exposes two main usage layers:

- `DataSetCreator`: high-level API for defining questions and generating datasets
- `DatasetGenerator`: lower-level engine that evaluates conditions and builds rows

`access_control_manager.py` adds role-based field visibility and update rules. Its integration is documented in a separate section below.

## Main Classes

### `QuestionDefinition`

Stores one question configuration.

Fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `possible_answers` | `List[Any]` | Allowed values for the field |
| `default_value` | `Any` | Fallback value when no condition matches |
| `dependency` | `Optional[str]` | Condition that decides whether the field is active |
| `copy_from` | `Optional[str]` | Source field to copy from |
| `skip` | `bool` | If `True`, the field is always `SKIP` |
| `if_conditions` | `List[Dict[str, Any]]` | Ordered condition-to-value mappings |
| `group` | `Optional[str]` | Group name used for permutation coverage |
| `weights` | `Optional[List[float]]` | Sampling weights aligned to `possible_answers` |
| `generation_strategy` | `str` | `random`, `weighted_random`, `exhaustive`, or `pairwise` |
| `ui_role` | `str` | `input`, `calculation`, or `display` |

### `LogicRuleDefinition`

Stores one derived-field rule set.

Fields:

| Field | Type | Meaning |
| --- | --- | --- |
| `target` | `str` | Output field name |
| `rules` | `List[Dict[str, Any]]` | Ordered condition-to-value mappings |
| `possible_values` | `List[Any]` | Allowed output values |
| `default` | `Any` | Fallback value if no rule matches |
| `position` | `Optional[int]` | Optional insertion index in column order |
| `ui_role` | `str` | `input`, `calculation`, or `display` |

### `ConditionExpression`

Wrapper for condition strings. Used by `ConditionBuilder`.

Helpers:

- `and_(other)`
- `or_(other)`
- `not_()`

### `ConditionBuilder`

Preferred helper for building conditions safely.

Available builders:

- `equals(field, value)`
- `not_equals(field, value)`
- `greater_than(field, value)`
- `less_than(field, value)`
- `greater_equal(field, value)`
- `less_equal(field, value)`
- `and_conditions(*conditions)`
- `or_conditions(*conditions)`
- `not_condition(condition)`

Example:

```python
from data_set_creator_and_generator import ConditionBuilder

adult = ConditionBuilder.greater_equal("age", 18)
premium = ConditionBuilder.equals("membership", "premium")
rule = ConditionBuilder.and_conditions(adult, premium)
```

## Condition Syntax

Conditions are evaluated through a validated AST-based evaluator.

Supported comparison operators:

- `==`
- `!=`
- `<`
- `<=`
- `>`
- `>=`
- `is`
- `is not`
- `in`
- `not in`

Supported logical operators:

- `AND` or `and`
- `OR` or `or`
- `NOT` or `not`

Supported values:

- strings
- integers
- floats
- booleans
- `None`
- tuples
- lists

Important rules:

- parentheses must be balanced
- field names must not be quoted
- quoted strings are allowed
- numeric comparisons are validated against declared answer types
- undefined referenced fields raise validation errors
- blank conditions and `"default_value"` are treated as non-expressions

Examples:

```python
"age >= 18 AND country == 'CA'"
"membership in ['gold', 'platinum']"
"NOT (status == 'inactive')"
```

## Recommended API: `DataSetCreator`

### Lifecycle

Typical flow:

1. Create `DataSetCreator()`
2. Optionally call `set_seed(seed)`
3. Add questions
4. Add logic fields
5. Optionally call `validate()`
6. Generate/export dataset

Example:

```python
from data_set_creator_and_generator import DataSetCreator, ConditionBuilder

creator = DataSetCreator()
creator.set_seed(42)

creator.add_base_question("age", [16, 18, 30, 65])
creator.add_base_question("membership", ["free", "premium"])

creator.add_conditional_question(
    "age_group",
    if_conditions=[
        {ConditionBuilder.less_than("age", 18): "minor"},
        {"default_value": "adult"},
    ],
    possible_answers=["minor", "adult"],
    ui_role="calculation",
)

creator.add_logic(
    "discount",
    rules=[
        {ConditionBuilder.equals("membership", "premium"): "10%"},
        {"default_value": "0%"},
    ],
    possible_values=["0%", "10%"],
    ui_role="display",
)

dataset = creator.generate_dataset_with_generator("output.csv")
```

## Question APIs

### `add_question(...)`

Power-user method that supports all question behaviors.

Signature summary:

```python
add_question(
    key,
    possible_answers=None,
    default_value=None,
    dependency=None,
    copy_from=None,
    skip=False,
    if_conditions=None,
    group=None,
    weights=None,
    generation_strategy="random",
    ui_role="input",
)
```

Parameters:

| Parameter | Meaning |
| --- | --- |
| `key` | Unique field name |
| `possible_answers` | Explicit answer list; required unless values can be inferred from `if_conditions` |
| `default_value` | Fallback value; must be inside `possible_answers` when provided |
| `dependency` | Condition controlling whether the field should be populated or set to `SKIP` |
| `copy_from` | Already-defined source field to copy from |
| `skip` | Marks the field as permanently `SKIP` |
| `if_conditions` | Ordered list like `[{condition: value}, {"default_value": fallback}]` |
| `group` | Group name for combination coverage |
| `weights` | Sampling weights aligned with `possible_answers` |
| `generation_strategy` | `random`, `weighted_random`, `exhaustive`, `pairwise` |
| `ui_role` | `input`, `calculation`, `display` |

Validation and behavior:

- question names must be unique across normal fields and logic targets
- `dependency` and `copy_from` cannot both be set
- `skip=True` cannot be combined with `dependency` or `copy_from`
- `copy_from` source must already exist
- `copy_from` source cannot be a logic/calculation target
- `weighted_random` requires `weights`
- if `weights` are provided with `generation_strategy="random"`, the strategy is promoted to `weighted_random`
- skipped questions cannot use `weighted_random`, `exhaustive`, or `pairwise`
- `weights` length must match `possible_answers`
- weights must be non-negative and not all zero
- `default_value` is inferred from the first answer if omitted
- when `possible_answers` is omitted, values are inferred from `if_conditions` plus `default_value`

### Helper Methods

Use these for clearer configuration:

#### `add_base_question(...)`

Adds a normal field with direct answer choices.

Options:

- `key`
- `possible_answers`
- `default_value`
- `group`
- `weights`
- `generation_strategy`
- `ui_role`

#### `add_conditional_question(...)`

Adds a field whose value comes from ordered `if_conditions`.

Options:

- `key`
- `if_conditions`
- `possible_answers`
- `default_value`
- `dependency`
- `group`
- `ui_role`

#### `add_copy_question(...)`

Adds a field that mirrors another already-defined field.

Options:

- `key`
- `possible_answers`
- `copy_from`
- `default_value`
- `ui_role`

#### `add_skipped_question(...)`

Adds a field that is always set to `SKIP`.

Options:

- `key`
- `possible_answers` defaulting to `["SKIP"]`
- `ui_role` defaulting to `display`

## Logic API

### `add_logic(...)`

Defines derived fields after the base dataset is generated.

Signature summary:

```python
add_logic(
    target_question,
    rules,
    possible_values=None,
    default_value=None,
    position=None,
    ui_role="calculation",
)
```

Parameters:

| Parameter | Meaning |
| --- | --- |
| `target_question` | Output column name |
| `rules` | Ordered list of `{condition: outcome}` mappings plus optional `{"default_value": x}` |
| `possible_values` | Optional allowed outputs; inferred from rule outcomes if omitted |
| `default_value` | Fallback output |
| `position` | Optional insertion index in `question_order` |
| `ui_role` | `input`, `calculation`, `display` |

Rules:

- logic target names cannot collide with existing question names
- every referenced field must already be defined
- all conditions must pass type compatibility checks
- conditions must have balanced parentheses
- if `possible_values` is omitted, they are inferred from seen rule outcomes
- `default_value` is appended to `possible_values` if missing

## Generation Strategies

### `random`

Uses `random.Random(seed).choice(...)`.

### `weighted_random`

Uses `random.Random(seed).choices(..., weights=...)`.

Requirements:

- `weights` must be present
- `weights` length must match answers

### `exhaustive`

Cycles through answer choices by row index so each answer is sampled repeatedly in order.

### `pairwise`

Handled during post-processing. The generator adds extra rows so every pair of fields marked `pairwise` covers all pair combinations.

## UI Roles

Used by `build_ui_payload(...)`.

| Role | Meaning |
| --- | --- |
| `input` | Direct user-editable field values |
| `calculation` | Derived/calculated outputs |
| `display` | Extra presentation-only fields |

## Internal Matrix Model

`DataSetCreator` compiles question definitions into a lower-level matrix consumed by `DatasetGenerator`.

Possible matrix forms:

- `"REQ"`: required field with direct sampling
- `"SKIP"`: always `SKIP`
- `{"D": condition}`: dependency-gated field
- `{"CP": source}`: copied field
- `{"IF": [...], "default": value}`: conditional mapping
- `{"D": condition, "IF": [...], "default": value}`: dependency plus conditional mapping

## Dataset Generation Pipeline

High-level generation in `DataSetCreator` runs this sequence:

1. validate logic rules
2. build `DatasetGenerator`
3. run dependency sampling test with `generate_test_set(num_samples=500)`
4. generate raw rows
5. fill missing values and apply `SKIP`
6. reduce the dataset to preserve coverage
7. apply group permutations
8. apply logic rules
9. reorder columns using `question_order`

Primary methods:

### `generate_dataset_with_generator(...)`

Recommended main export path.

Options:

- `filename=None`: auto-generates `complex_logic_dataset_<YYYYMMDD>.csv`
- `helper_info=False`: enables logging
- `acm=None`: optional `AccessControlManager`
- `num_rows=500`: number of initial rows before reduction/permutation

Behavior:

- exports CSV
- if `acm` is provided, also exports companion ACM JSON
- returns the final `DataFrame`

### `generate_dataset(...)`

Returns a generated `DataFrame` without exporting ACM data.

### `generate_dataset_with_acm(...)`

Returns a generated `DataFrame` and accepts an optional ACM object, but unlike `generate_dataset_with_generator(...)` it does not export files by itself.

### `generate(...)`

Shortcut to `generate_dataset_with_generator(...)`.

## Dataset Post-Processing APIs

### `generate_test_set(num_samples=500)`

Randomly samples contexts to confirm each dependency can evaluate to `True` at least once.

### `fill_missing_data(df)`

Behavior:

- fills required (`REQ`) fields with defaults
- enforces dependency-based `SKIP`
- copies values for `CP` fields
- fills remaining nulls with `SKIP`

### `reduce_to_minimum(df)`

Tries to keep at least one row for each target answer and dependency path.

### `apply_group_permutations(df)`

Ensures full answer-combination coverage inside each named group. After group expansion it also re-checks dependency-based `SKIP`. It then applies pairwise coverage for `pairwise` questions.

## Import, Export, and Utility APIs

### `export_dataset(dataset, filename)`

Writes CSV.

### `export_to_excel(dataset, filename, acm=None)`

Creates an Excel workbook with:

- `dataset` sheet
- `coverage` sheet
- `acm_report` sheet when ACM is supplied
- one sheet per role when ACM is supplied

### `export_pre_generator_data(filename="pre_generator_output.csv")`

Exports a configuration summary before row generation.

### `load_dataset(filename)`

Loads CSV into a `DataFrame`.

### `filter_dataset(dataset, criteria)`

Case-insensitive exact-match filtering on column values.

### `build_ui_payload(dataset, criteria=None, record_index=0, include_all_matches=True)`

Returns:

```python
{
    "criteria": {...},
    "match_count": ...,
    "selected_index": ...,
    "field_values": {...},
    "calculations": {...},
    "display_fields": {...},
    "selected_record": {...},
    "records": [...],  # optional
}
```

Field routing is based on `ui_role`.

### `filter_csv_to_ui_payload(filename, criteria, record_index=0, include_all_matches=True)`

Loads a CSV, filters it, and returns the UI payload.

### `save_profile(filename)` / `load_profile(filename)`

Persist and reload:

- seed
- questions
- logic rules
- question order
- groups through question definitions
- weights
- generation strategies
- UI roles

### `generate_coverage_report(dataset)`

Returns:

- `answer_coverage`
- `logic_coverage`
- `dependency_coverage`

### `validate()`

Returns:

```python
{
    "errors": [...],
    "warnings": [...],
}
```

Checks include:

- undefined question references
- invalid logic conditions
- missing weights for `weighted_random`
- incompatible skipped-field strategies
- copying from logic fields
- bad weight lengths or all-zero weights
- one-field groups

### `get_question_order()`

Returns the current column order.

### `get_defined_questions()`

Returns every defined question and logic target name.

### `reset()`

Clears the current creator state.

## Direct Engine API: `DatasetGenerator`

Use this only if you already have a compiled matrix and answer map.

Constructor:

```python
DatasetGenerator(
    matrix,
    answers_per_question,
    helper_info=False,
    groups=None,
    seed=42,
    question_configs=None,
)
```

Inputs:

| Parameter | Meaning |
| --- | --- |
| `matrix` | Matrix produced by `DataSetCreator._build_matrix()` |
| `answers_per_question` | Answer list per field |
| `helper_info` | Verbose logging |
| `groups` | Group definitions |
| `seed` | Random seed |
| `question_configs` | Optional question definitions used for advanced strategies |

Important direct methods:

- `evaluate_condition(condition, context)`
- `generate_dataset(num_rows=500)`
- `fill_missing_data(df)`
- `reduce_to_minimum(df)`
- `apply_group_permutations(df)`
- `generate_large_dataset(num_rows=10000, chunk_size=1000)`
- `generate_with_acm(filename=None, acm=None, num_rows=500)`
- `save_to_csv(df, filename)`
- `load_from_pickle(filename="dataset.pkl", helper_info=False)`
- `generate(filename="output.csv", acm=None, num_rows=500)`

Direct `DatasetGenerator.generate_with_acm(...)` behavior is different from `DataSetCreator`:

- it appends permission metadata columns like `<field>_permission_<role>`
- it does not build the JSON ACM record view used by `DataSetCreator.export_acm_view(...)`

## Access Control Manager Integration

`access_control_manager.py` defines `AccessControlManager`, which integrates with the dataset generator in three main ways:

1. field-level and role-level permission definition
2. record-level conditional permission overrides
3. export of ACM-aware dataset views

### Permission Types

Supported constants:

- `AccessControlManager.PERMISSION_UPDATABLE`
- `AccessControlManager.PERMISSION_HIDDEN`
- `AccessControlManager.PERMISSION_VIEW_ONLY`

### ACM Setup API

#### `AccessControlManager(dataset_columns=None, name="acm")`

Options:

- `dataset_columns`: optional known column list for validation
- `name`: used by `save_permissions()` default filename generation

#### `add_role(role_name)`

Registers a role name.

#### `set_dataset_columns(columns)`

Registers dataset columns so permission validation can warn about unknown fields.

#### `set_permission(field_name, role, permission)`

Sets one normalized field/role permission entry.

Behavior:

- adds unknown roles automatically with a warning
- warns if `field_name` is not present in known dataset columns

#### `define_permissions(field_name, permissions)`

Bulk field permission definition.

Expected format:

```python
acm.define_permissions("salary", {
    AccessControlManager.PERMISSION_UPDATABLE: ["admin"],
    AccessControlManager.PERMISSION_VIEW_ONLY: ["manager"],
    AccessControlManager.PERMISSION_HIDDEN: ["guest"],
})
```

It rejects conflicting permissions for the same field/role pair.

#### `define_conditional_permission(field_name, role, condition, permission)`

Adds a record-level override evaluated against each row.

Example:

```python
acm.define_conditional_permission(
    "salary",
    "viewer",
    "department == 'IT'",
    AccessControlManager.PERMISSION_HIDDEN,
)
```

Condition evaluation reuses `DatasetGenerator.evaluate_condition(...)`.

### ACM Query API

#### `get_permissions(field_name, role, record=None)`

Returns the effective permission. If `record` is supplied, conditional rules are checked first.

#### `can_access(field_name, role, record=None)`

Returns `False` only when the effective permission is `hidden`.

#### `can_update(field_name, role, record=None)`

Returns `True` only when the effective permission is `updatable`.

#### `generate_permissions_report(role=None)`

Returns a `DataFrame` of base and conditional permissions.

#### `validate_permissions()`

Returns warnings for protected fields that are not present in `dataset_columns`.

### Integration Paths Inside `DataSetCreator`

#### 1. Companion ACM JSON export

Use:

```python
dataset = creator.generate_dataset_with_generator("employees.csv", acm=acm)
```

Outputs:

- `employees.csv`
- `acm_employees.json`

JSON structure:

- one top-level key per row, such as `record_0`
- `data`: the raw row dictionary
- `permissions`: permission map by role, then by protected column

Only columns present in `acm.permissions` are included in the permission map.

#### 2. Role-specific CSV export

Use:

```python
viewer_df = creator.export_role_specific_dataset("viewer", acm, "viewer.csv")
```

Behavior:

- generates a fresh dataset
- registers generated columns in ACM
- applies visibility rules with `apply_permissions_to_dataset(...)`
- writes a role-filtered CSV

Important implementation detail:

- if any row causes a column to be hidden for the selected role, that column is dropped from the entire exported dataset for that role

#### 3. Excel export with ACM sheets

Use:

```python
creator.export_to_excel(dataset, "bundle.xlsx", acm=acm)
```

Additional sheets:

- `acm_report`
- `role_<role>`

### Integration Path Inside `DatasetGenerator`

`DatasetGenerator.generate_with_acm(...)` is a separate legacy-style path:

- validates permissions
- adds metadata columns for protected fields and roles
- saves a single CSV

Example added columns:

- `salary_permission_admin`
- `salary_permission_viewer`

Use `DataSetCreator.generate_dataset_with_generator(..., acm=acm)` if you want the cleaner CSV-plus-JSON split.

## Practical Recommendations

- Prefer `DataSetCreator` over direct `DatasetGenerator` usage.
- Prefer helper methods like `add_base_question(...)` over raw `add_question(...)`.
- Prefer `ConditionBuilder` over hand-written condition strings.
- Call `validate()` before generation for non-trivial configurations.
- Use `ui_role` consistently if the dataset will feed a UI.
- Use ACM companion JSON when a UI needs per-record permission metadata.
- Use role-specific CSV export when a downstream consumer should only see allowed columns.
