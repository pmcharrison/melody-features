# Package structure migration plan

This document is a handoff plan for a follow-up agent conversation. The goal is
to continue cleaning up the package structure after the feature-definition split,
while preserving the public API and existing backwards-compatible import paths.

## Current assumptions

- `melody_features.features` remains the compatibility facade and public
  orchestration entry point.
- Category feature definitions have been split into explicit `*_features.py`
  modules.
- The package supports Python 3.9+, so do not introduce PEP 604 union syntax
  such as `str | None`; use `Optional[...]` or `Union[...]`.
- Keep commits scoped and reviewable. Prefer one subsystem per PR.
- Keep old import paths working with shims or re-exports.

## 1. Preserve public compatibility first

- Keep package-root exports in `melody_features/__init__.py` unchanged.
- Keep `melody_features.features` importable with the same public symbols.
- For any moved module that may be externally imported, leave a thin
  compatibility shim at the old path.

Example:

```python
# import_mid.py
from .io.midi import *
```

## 2. Create target package directories

Add package directories with `__init__.py` files:

```text
src/melody_features/feature_definitions/
src/melody_features/io/
src/melody_features/idyom/
src/melody_features/pipeline/
src/melody_features/utils/
```

Consider a later `corpus/` package, but be careful: `corpus.py` already exists,
so converting it into a package requires a staged rename and compatibility shim.

## 3. Move feature definition modules

Move existing category files into `feature_definitions/`.

Example target names:

```text
absolute_pitch_features.py -> feature_definitions/absolute_pitch.py
pitch_class_features.py -> feature_definitions/pitch_class.py
pitch_interval_features.py -> feature_definitions/pitch_interval.py
corpus_features.py -> feature_definitions/corpus.py
```

Update `features.py` facade imports to point at the new package.

Leave temporary shims at the old category-module paths:

```python
# absolute_pitch_features.py
from .feature_definitions.absolute_pitch import *
```

Run at minimum:

```bash
PYTHONPATH=src python -m pytest tests/test_feature_facade_exports.py tests/test_features_systematic.py -q
```

## 4. Move MIDI and IO helpers

Move:

```text
import_mid.py -> io/midi.py
```

Potential future moves:

```text
representations.py -> io/representations.py
```

Do not move `representations.py` in the first IO PR unless necessary. `Melody`
is central and widely imported.

Add a compatibility shim:

```python
# import_mid.py
from .io.midi import *
```

Run:

```bash
PYTHONPATH=src python -m pytest tests/test_corpus_import.py tests/test_partitura.py tests/test_module_setup.py -q
PYTHONPATH=src python -m pytest -q
```

## 5. Move IDyOM internals

Create:

```text
idyom/config.py
idyom/interface.py
idyom/runners.py
```

Move or split:

- `IDyOMConfig`
- IDyOM default config builders
- IDyOM corpus resolution helpers
- current `idyom_interface.py`
- `get_idyom_results`
- temporary MIDI/key-signature helpers
- retry and IDyOM analysis orchestration

Keep these import paths working:

- `melody_features.features.IDyOMConfig`
- `melody_features.IDyOMConfig`
- `melody_features.idyom_interface`

Run:

```bash
PYTHONPATH=src python -m pytest tests/test_idyom.py tests/test_idyom_setup.py tests/test_idyom_corpus_config.py -q
PYTHONPATH=src python -m pytest -q
```

## 6. Move general pipeline configuration

Create:

```text
pipeline/config.py
```

Move:

- `Config`
- `FantasticConfig`
- config validation helpers
- `_setup_default_config`
- `_validate_config`

Keep re-exports from:

- `melody_features.features`
- `melody_features.__init__`

Run:

```bash
PYTHONPATH=src python -m pytest tests/test_module_setup.py tests/test_corpus_features.py tests/test_idyom_corpus_config.py -q
PYTHONPATH=src python -m pytest -q
```

## 7. Move loading and corpus setup orchestration

Create:

```text
pipeline/loading.py
pipeline/corpus_setup.py
```

Move:

- `_load_melody_data`
- corpus-stat setup orchestration if not already isolated
- input path validation helpers

Keep compatibility re-exports in `features.py`.

Run:

```bash
PYTHONPATH=src python -m pytest tests/test_corpus_features.py tests/test_corpus_import.py tests/test_features_systematic.py -q
PYTHONPATH=src python -m pytest -q
```

## 8. Move per-melody processing and parallel orchestration

Create:

```text
pipeline/processing.py
```

Move:

- `process_melody`
- `_process_melodies_parallel`
- `_process_single_melody`
- worker argument construction

Keep wrappers/re-exports in `features.py`.

Run:

```bash
PYTHONPATH=src python -m pytest tests/test_timing_stats.py tests/test_features_systematic.py -q
PYTHONPATH=src python -m pytest -q
```

## 9. Move output and timing-stat construction

Create:

```text
pipeline/output.py
pipeline/timing.py
```

Move:

- `TIMING_STAT_CATEGORIES`
- `_init_timing_stats`
- `_get_category_display_name`
- header construction
- row flattening
- DataFrame writing/export logic

Run:

```bash
PYTHONPATH=src python -m pytest tests/test_timing_stats.py tests/test_features_systematic.py -q
PYTHONPATH=src python -m pytest -q
```

## 10. Move small cross-cutting utilities

Only use `utils/` for genuinely small, cross-cutting helpers. Avoid creating a
large dumping ground.

Possible files:

```text
utils/logging.py
utils/validation.py
utils/paths.py
```

Possible moves:

- `_setup_logger`
- `_validate_viewpoints`
- `_check_is_monophonic`
- tiny path helpers

## 11. Reassess `features.py`

After the moves above, `features.py` should mostly contain:

- public imports and re-exports
- `get_all_features` wrapper
- backwards-compatible private helper aliases

At that point decide whether to:

- keep `features.py` permanently as the compatibility facade, or
- gradually deprecate direct imports from it.

## 12. Later phase: remove category decorators

Do not combine this with the package-structure move.

Possible later approach:

- use module location or explicit module manifests as category metadata
- keep source/citation decorators if still useful
- update registry discovery to use module manifests instead of category
  decorators
- remove `@absolute`, `@interval`, `@timing`, etc. in a separate, heavily
  tested PR

## General implementation checklist for each PR

1. Create one focused branch.
2. Move one subsystem only.
3. Add shims/re-exports before deleting old imports.
4. Run targeted tests first.
5. Run the full suite:

   ```bash
   PYTHONPATH=src python -m pytest -q
   ```

6. Verify no root-level corpus-stat artifacts are generated.
7. Verify no added `|` union syntax in changed Python files.
8. Commit, push, and open/update a draft PR.
