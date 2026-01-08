# arXiv LaTeX Normalization & Reference Matching Pipeline

This project normalizes arXiv LaTeX sources, merges cross-version references, auto-labels BibTeX–arXiv pairs, and trains a reference-matching model.

## Features

- **Multi-version analysis**: Scan all LaTeX versions of each paper, build a hierarchy (chapter/section/paragraph/sentence, figures, tables, formulas) and content hashes for deduplication.
- **Normalize & merge references**: Unify BibTeX across versions, deduplicate by content, export a cleaned `refs.bib`.
- **Automatic labeling**: Match BibTeX keys to `references.json` via URL/DOI/eprint rules, string normalization and fuzzy matching (rapidfuzz); write [labels/auto_label.json](labels/auto_label.json).
- **Train & evaluate matching**: Create features from title/author/year/embeddings (SentenceTransformer), train RF/GB/LR models, evaluate with MRR, and write `pred.json` per paper.
- **Parallel processing**: Batch multiple papers via ThreadPool with version/element statistics.

## Environment Setup

### Python Version

- **Python 3.7+** (Python 3.8 or higher recommended)

### Required Python Packages

Install the required packages using `requirements.txt` (recommended):

```bash
pip install -r requirements.txt
```

Or install packages individually:

```bash
pip install bibtexparser rapidfuzz python-Levenshtein sentence-transformers scikit-learn numpy pandas nltk
```

Or using conda:

```bash
conda install -c conda-forge bibtexparser rapidfuzz python-Levenshtein sentence-transformers scikit-learn numpy pandas nltk
```

Or run cell install in the Jupyter notebook:
```bash

!pip install bibtexparser rapidfuzz python-Levenshtein sentence-transformers scikit-learn numpy pandas nltk
```

## Project Structure

After running the notebook, you'll have the following files:

```
23127238/
  ├── src/
    │    ├── paper_processing.ipynb      # Normalize LaTeX, export hierarchy & refs
    │    ├── data_labelling.ipynb        # Auto-label BibTeX → arXiv ID
    │    └── reference_matching.ipynb    # Train/evaluate, generate predictions
  ├── labels/
  │    ├── manual_label.json            # Handmade labels (available)
  │    └── auto_label.json              # Labels automatically generated from data_labelling
  └── 23127238_output/                  # Processed outputs
	├── 2304-14607/
    │    ├── metadata.json           # Copied from crawler input
    │    ├── references.json         # Copied from crawler input
    │    ├── hierarchy.json          # Content hierarchy tree
    │    ├── refs.bib                # Deduplicated & cleaned BibTeX (if LaTeX parse succeeds)
    │    └── pred.json               # Matching predictions (step 3, if labeled)
    └── ...                          # One folder per paper
```

> Expected input: each arXiv paper folder `YYMM-NNNNN/` containing `metadata.json`, `references.json`, and `tex/YYMM-NNNNNvN/` (LaTeX sources extracted by the crawler).

## Output Conditions

- `hierarchy.json`: always written if at least one `.tex` version is parsed (from [src/paper_processing.ipynb](src/paper_processing.ipynb)).
- `refs.bib`: present only when LaTeX parsing succeeds and source contains BibTeX; if no main `.tex` or no BibTeX, the paper folder is created without `refs.bib`.
- `pred.json`: created only for papers that appear in [labels/manual_label.json](labels/manual_label.json) or [labels/auto_label.json](labels/auto_label.json). Currently ~1505 papers (≈10% + 5 manual) have labels, so only those get `pred.json` after running [src/reference_matching.ipynb](src/reference_matching.ipynb).

## Workflow

1) **Normalize LaTeX & extract references** — open [src/paper_processing.ipynb](src/paper_processing.ipynb):
        - Set `FOLDER` to the input folder and `OUTPUT_FOLDER` (default `<FOLDER>_output`).
        - Two built-in modes:
                - *For one paper*: uncomment the sample block, set `FOLDER = ".../YYMM-NNNNN"`, run the related cells.
                - *For all papers*: keep the final block, set parent `FOLDER` and `OUTPUT_FOLDER`, run the notebook end-to-end.
        - The notebook locates the main `.tex`, builds the hierarchy, merges references, writes `hierarchy.json`/`refs.bib`, and copies `metadata.json`/`references.json` into the output folder.
        - Tune parallelism via `max_workers` in `process_all()` (default 4).

2) **Auto-label BibTeX → arXiv** — open [src/data_labelling.ipynb](src/data_labelling.ipynb):
        - Set `base_dir` (e.g., `23127238_output`), `start_folder`, `papers_to_check`.
        - `papers_to_check` is the maximum number of papers to auto-label; increase/decrease to control label coverage.
        - Run the notebook; outputs to [labels/auto_label.json](labels/auto_label.json). Logs show skipped papers missing `references.json` or `.bib`.

3) **Train & evaluate matching** — open [src/reference_matching.ipynb](src/reference_matching.ipynb):
        - Set `OUTPUT_DIR` to the results folder; ensure [labels/manual_label.json](labels/manual_label.json) and [labels/auto_label.json](labels/auto_label.json) exist.
        - The notebook loads BibTeX and `references.json`, creates train/val/test splits from labels, generates features, scales data, trains RF/GB/LR, selects the best model (train accuracy), and computes MRR on test.
        - Each labeled paper gets a `pred.json` (top-k candidates + ground truth) in its folder within `OUTPUT_DIR`.

## Example Usage

### Run a single paper (paper_processing)

In [src/paper_processing.ipynb](src/paper_processing.ipynb), enable the “For one paper” block and set:

```python
FOLDER = "23127238/2304-14745"
OUTPUT_FOLDER = "23127238_output"

batch_processor = BatchProcessor(FOLDER, OUTPUT_FOLDER)
paper = batch_processor.process_paper(Path(FOLDER))
```

### Run all papers in a folder

Keep the “For all papers” block and set:

```python
FOLDER = "23127238"              # folder containing many YYMM-NNNNN songs
OUTPUT_FOLDER = "23127238_output"

batch_processor = BatchProcessor(FOLDER, OUTPUT_FOLDER)
stats = batch_processor.process_all(max_workers=4)
```

### Increase/decrease auto-labeled papers (data_labelling)

In [src/data_labelling.ipynb](src/data_labelling.ipynb), set:

```python
base_dir = Path("23127238_output")
papers_to_check = 2000  # Example: Scan 2000 articles
```

### Generate predictions (reference_matching)

In [src/reference_matching.ipynb](src/reference_matching.ipynb), once labels and BibTeX are available:

```python
OUTPUT_DIR = Path("23127238_output")
predictions = generate_predictions(best_model, scaler,
																	 all_bibtex[paper_id], all_references[paper_id],
																	 top_k=5)
```

## Data File Formats

### metadata.json Structure

```json
{
    "arxiv_id": "2305-02001",
    "paper_title": "Surreal substructures",
    "authors": [
        "Vincent Bagayoko",
        "Joris van der Hoeven"
    ],
    "submission_date": "2023-05-03",
    "revised_dates": [],
    "publication_venue": null,
    "latest_version": 1,
    "categories": [
        "math.LO"
    ]
}
```

### references.json Structure

```json
{
    "2402-15800": {
        "paper_title": "Sign sequences of log-atomic numbers",
        "authors": [
            "Vincent Bagayoko"
        ],
        "submission_date": "2024-02-24",
        "semantic_scholar_id": "2d9e48266edf82c418850d3096e2db2059941625"
    },
    ...
}
```

### hierarchy.json Structure (simplified)

```json
{
	"elements": {
        "2304-14608|73df1fea2e4dfb72": "\\document{Document}",
        "2304-14608|78d5c28e2658558e": "\\abstract{Abstract}",
		"2304-14608|0ca043c5547e45cd": "Photons propagating in water undergo absorption and scattering processes."
	},
	"hierarchy": {
        "1": {
            "2304-14608|78d5c28e2658558e": "2304-14608|73df1fea2e4dfb72",
            "2304-14608|7300ea78b785996c": "2304-14608|78d5c28e2658558e",
            "2304-14608|a71a8da791b6d284": "2304-14608|78d5c28e2658558e",
            "2304-14608|a57b4b8c44465058": "2304-14608|78d5c28e2658558e"
        },
        "2": {
            "2304-14608|78d5c28e2658558e": "2304-14608|73df1fea2e4dfb72",
            "2304-14608|7300ea78b785996c": "2304-14608|78d5c28e2658558e",
            "2304-14608|a71a8da791b6d284": "2304-14608|78d5c28e2658558e"
        }
    }
}
```

### refs.bib

- Aggregated BibTeX merged and deduplicated across versions. Each entry follows a standard form:

```bibtex
@article{IceCube:2013,
    author = {Aartsen, M. G. and others},
    collaboration = {IceCube},
    title = {{Evidence for High-Energy Extraterrestrial Neutrinos at the IceCube Detector}},
    eprint = {1311.5238},
    archiveprefix = {arXiv},
    primaryclass = {astro-ph.HE},
    doi = {10.1126/science.1242856},
    journal = {Science},
    volume = {342},
    pages = {1242856},
    year = {2013},
}
```

### labels/manual_label.json & labels/auto_label.json

```json
{
    "2304-14610": {
        "Yang_2016_CVPR": "1511-06523",
        "Pool-3FC": "1904-01382",
        "PIAA": "2203-16754",
        "liang2022semantically": "2112-06451"
    },
    "2304-14656": {
        "pbl": "1810-04444",
        "qatten": "2002-03939"
    }
}
```

### pred.json Structure

```json
{
    "partition": "test",
    "groundtruth": {
        "IceCube:2013": "1311-5238",
        "IceCube:2018": "1807-08816"
    },
    "prediction": {
        "IceCube:2013": [
            "1311-5238",
            "2305-01967",
            "2303-02911",
            "2302-05032",
            "2211-09972"
        ],
        "IceCube:2018": [
            "1807-08816",
            "2305-01967",
            "2303-02911",
            "2302-05032",
            "2211-09972"
        ]
    }
}
```

## Troubleshooting

- Missing `refs.bib`: verify the `tex/` folder has a main `.tex` and BibTeX; if the parser can’t find the main file or no BibTeX entries exist, the output will lack `refs.bib`.
- Missing `pred.json`: generated only for labeled papers (manual or auto). Check [labels/auto_label.json](labels/auto_label.json) and [labels/manual_label.json](labels/manual_label.json).
- Missing [labels/auto_label.json](labels/auto_label.json): increase `papers_to_check` or adjust `start_folder` in [src/data_labelling.ipynb](src/data_labelling.ipynb).
- `sentence-transformers` requires Torch: install `pip install torch --extra-index-url https://download.pytorch.org/whl/cpu` (or GPU variant).
- NLTK tokenizer missing `punkt`: run `python -c "import nltk; nltk.download('punkt')"`.
- Out-of-memory on large batches: reduce `max_workers` in `process_all()` or split the input folder.
