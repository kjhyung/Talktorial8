# How I completed Talktorial T008 — working notes

Notes on building a runnable version of
[TeachOpenCADD T008 · Query the PDB](https://projects.volkamerlab.org/teachopencadd/talktorials/T008_query_pdb.html)
from scratch in an empty repository. This is the reasoning behind the notebook, not a
replacement for it: `talktorial.ipynb` is the artefact, this file is the "why".

---

## 1. The starting point

The repository contained nothing but `.git`. So "complete this" could not mean "fill in
the blanks" — it meant producing the whole notebook: theory text, practical steps, and
code that actually runs against the live PDB.

Before writing a single cell I wrote down three constraints for the result:

1. It has to be recognisably the published talktorial — same narrative, same criteria,
   same story arc from "how many EGFR structures exist" to "here are four aligned ligands".
2. It has to run end to end, not just look right.
3. Any deviation from the published code must be a consequence of the environment, and
   must be written down in the notebook itself, not only in a commit message.

## 2. Getting the source material instead of retyping it

The rendered HTML page is a few megabytes (all the outputs are inlined), so I pulled the
notebook source from the TeachOpenCADD GitHub repository and dumped every cell's source
to a text file. Reading 82 cells as one flat list first was useful: it showed the
structure (theory → counts → filters → metadata → ligands → alignment) and it showed that
the modern version has **no exercise cells**, only narrative plus code. That decided the
deliverable: reproduce and verify, not "solve tasks".

I also fetched `images/protein-ligand-complex.png` and the talktorial's `requirements.txt`,
because the rendered notebook references that figure and the requirements file is a useful
starting point for pins.

## 3. Auditing the environment before writing code

This is where most of the time went, and it was worth it. The sandbox offers Python 3.15
as default and 3.14 as a secondary interpreter.

| Attempt | Result | Consequence |
|-|-|-|
| `pip install biotite` (3.15) | Fails: `biotraj` compiles a C extension, MSVC build tools missing | No `biotite` on 3.15 |
| `pip install biotite` (3.14) | Same failure | Same |
| `pip install --no-deps biotite` (3.14) | Works; `biotite.database.rcsb` imports fine | Only the query module is needed, no trajectories |
| `pip install rdkit` (3.14) | Works (2026.09) | Use 3.14 throughout |
| `pip install MDAnalysis biopandas` (3.14) | Works (2.11) | Superposition is possible |
| `pip install opencadd` | No distribution on PyPI at all | Cannot be used |
| `pip install git+…opencadd@5f9e18f` | Build fails: `versioneer` calls `configparser.SafeConfigParser`, removed in Python 3.12+ | Cannot be used |

Two lessons here:

- **`--no-deps` is not a hack if you know which submodule you need.** `biotraj` only
  powers trajectory I/O; `biotite.database.rcsb` needs `numpy`/`requests`/`msgpack`, which
  install as wheels. Installing the package without its optional heavy dependency was
  enough for the queries the talktorial performs.
- **A dead dependency is a design decision, not a blocker.** `opencadd` is the superposition
  library the talktorial uses, but it only wraps `MDAnalysis`. Reimplementing its one call
  in `MDAnalysis` directly keeps the science identical and removes an unbuildable package
  from the chain. That is the only substantive deviation in the notebook, and the cell
  text says so explicitly.

Writing prose around a package that cannot be installed would have been the easy but
dishonest route, so the theory/reference sections were adjusted too.

## 4. Probing the live APIs before committing to code

The PDB migrated to a new search API, and half of `pypdb`'s older examples are stale, so
I checked each endpoint with throwaway scripts before putting anything in a notebook:

- Search API (`search.rcsb.org/rcsbsearch/v2/query`) — reachable, 394 EGFR entries.
- The six `biotite` field queries — all accepted, and the composite returns 110 entries.
  Importantly, the frozen set used later in the talktorial (5UG9, 5HG8, 5UG8, 5UGC) is
  actually among those 110 hits, so the notebook's story stays coherent.
- `pypdb.describe_pdb` — returns the expected `entry` / `rcsb_entry_info` structure.
- The GraphQL non-polymer query — 200 OK.
- The `chemcomp` REST endpoint — 200 OK, but see the descriptor-key trap below.

Deciding the field names, the return shapes and the failure modes up front meant the
notebook cells could be written once instead of debugged in place.

## 5. A dry run of the whole pipeline

Before generating the notebook I ran the entire pipeline as a plain script: download the
four structures, select protein + ligand, superpose, extract ligands, draw the molecules.
This exposed two real bugs that reading the source would never have shown.

### Bug 1 — the "largest ligand" test silently picks glycerol

`chem_comp.formula_weight` comes back as a **string** (`"445.494"`). The original
`max(ligands.items(), key=lambda kv: kv[1]["Molecular weight"])` therefore compares
strings, and `"92.094" > "445.494"` lexicographically. The notebook would have selected
glycerol (GOL) as the ligand for every structure and the error would have looked like a
data problem, not a type problem. Fix: cast to `float` in the metadata helper.

### Bug 2 — the superposition fails on 5HG8

`align.alignto` raised

```
SelectionError: Reference and trajectory atom selections do not contain the same number of atoms:
atoms:    N_ref=286, N_traj=306
```

PDB 5HG8 was crystallised with a longer construct (residues ~700–999, 306 Cα) than the
other three (~702–985, 286 Cα). This is exactly the case `opencadd` handled by aligning
sequences first. The fix is a few lines: intersect residue numbers across structures,
superpose those Cα atoms only, and apply the transformation to the whole complex via
`subselection`. After the fix the RMSDs are 25.34 Å → 2.70 Å for 5HG8 and ≤0.2 Å for the
others, which is the sanity check that the alignment actually did something.

### Bug 3 — `nglview` and overwritten files

Passing `AtomGroup` objects to `view.add_component` failed with

```
ValueError: Inconsistency in file 'data/5UG9.pdb': The number of atoms (2330) in
trajectory frame 0 differs from the number of atoms (2664) in the corresponding topology.
```

The cause is subtle: the complex subset is written back to `data/5UG9.pdb`, overwriting
the file the `Universe` was built from, so the in-memory topology (2664 atoms) no longer
matches the file on disk. `nglview` re-reads the file and trips over it. Passing the file
*paths* instead of the objects resolves it, and also makes the viewer code independent of
MDAnalysis internals.

## 6. Writing the notebook

Structure decisions:

- **Cell-for-cell parity** with the published notebook, plus a `Requirements` cell that
  explains the environment in place of the Colab install cell.
- **No `_dh` magic.** `HERE = pathlib.Path.cwd()` keeps the notebook runnable from any
  Jupyter frontend and from `nbconvert` without IPython's input-history hook.
- **`data/` created by the notebook** and gitignored, so a fresh clone just works.
- **Outputs kept in the committed notebook.** They are the evidence the notebook ran, and
  they match what the reader sees on the website.
- **Two `render_image` cells dropped.** They need a live frontend; the interactive widget
  cells are kept, with a note that they only render in Jupyter.

## 7. The verification loop, and what it caught

I generated the notebook from a small builder script (so cells could be rewritten without
hand-editing JSON), executed it headlessly, and then inspected the result rather than
trusting nbconvert's exit code:

```bash
python -m jupyter nbconvert --to notebook --execute --inplace \
    --ExecutePreprocessor.kernel_name=python3.14 \
    --ExecutePreprocessor.timeout=900 talktorial.ipynb
```

Then a script walks the notebook JSON and reports any `output_type == "error"` plus the
text of the cells I care about, and the generated artefacts are checked on disk.

The first execution passed with **zero errors** — and was still wrong: the ligand CSV had
`InChI` and `InChIKey` empty for all four ligands. Reading the API response showed that
`rcsb_chem_comp_descriptor` is now a **dict** keyed by descriptor type (`InChI`,
`InChIKey`, `SMILES`, `SMILES_stereo`), while the notebook's fallback branch still looked
for the old lowercase keys (`in_ch_i`, `in_ch_ikey`, `smilesstereo`). After the fix,
`MJLFLAORJNTNDV-CHWSQXEVSA-N` appears as expected.

This is the part I would emphasise to anyone repeating the exercise: **"no error output"
and "correct output" are different assertions.** The check that found this was reading
the produced CSV, not reading the execution log.

Final state of the checks:

- nbconvert exit status 0, zero error outputs across 34 code cells.
- 110 PDB matches; top-6 by resolution; frozen set 5UG9 / 5HG8 / 5UG8 / 5UGC.
- Ligand table: 8AM, 634, 8BP, 8BS with molecular weights 445.494 / 377.4 / 415.468 / 417.441
  and no missing InChI, InChIKey or SMILES.
- `data/` contains 4 complex PDBs, 4 `*_lig.pdb` files and `PDB_top_ligands.csv`.

## 8. What is not verified

- The `nglview` widgets only render in a live Jupyter session; headless execution stores
  widget state but proves nothing about the rendering.
- The yearly count plot is only checked for "it executes"; it depends on the API returning
  historical counts.
- Numbers like "110 matches" are a snapshot of the PDB as of the run. The deposition-date
  cutoff keeps the selected structures stable, but counts will drift.

## 9. Reproducing it

```bash
pip install -r requirements.txt   # Python 3.14
jupyter lab talktorial.ipynb
```

Then "Run All". The first run downloads ~110 metadata records and 4 structures; later runs
reuse `data/`.

## 10. Order of operations, condensed

1. Read all notebook cells from source, not from the rendered page.
2. Audit the interpreter and dependency stack before writing code.
3. Probe every external endpoint with scratch scripts.
4. Dry-run the full pipeline as a script; fix type/API/geometry surprises there.
5. Generate the notebook with parity to the original, documenting deviations inline.
6. Execute headlessly, then inspect outputs and artefacts — not just the exit code.
7. Clean up the temporary builder script and the scratch files.
