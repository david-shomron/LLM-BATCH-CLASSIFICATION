# LLM-based occupation classification

A taxonomy-grounded system for classifying open-ended Hebrew survey responses into a standard four-digit Israeli occupation code. The proposed **retrieve → rerank → select** pipeline combines semantic retrieval with context-aware language-model decisions.

The business aim is to reduce manual occupation coding and support timely labour-market statistics. This implementation demonstrates the approach using a dummy survey dataset; no model is trained on that dataset.

## Proposed model: 50 → 10 → 1

| Step | Model and input | Output |
| --- | --- | --- |
| Prepare knowledge base | Three taxonomy spreadsheets and occupation definitions, tasks and exclusions from the DOCX | Structured taxonomy text and enriched occupation cards |
| 1. Retrieve | **multilingual-E5-small** embeds the survey query and structured taxonomy text; cosine similarity ranks distinct codes | 50 candidate codes |
| 2. Rerank | **BGE-reranker-v2-m3** scores each query–occupation-card pair independently | Ten leading candidates |
| 3. Select | **Qwen3-4B-Instruct-2507** compares the ten cards under occupational coding rules | One candidate code |
| Evaluate | Compare candidates and final predictions with eligible reference labels | Retrieval Recall@50, shortlist Recall@10, final accuracy and Macro F1 |

The knowledge base is built before retrieval, independently of survey queries. E5 uses the spreadsheet-derived `text_context`; the reranker and selector use cards enriched with DOCX evidence. The selector returns structured JSON, and the code validates that its choice belongs to the shortlist.

This design can use a maintained taxonomy without retraining a classifier for every occupation. Its limits include missed retrieval candidates, ambiguous survey responses and incorrect model decisions. A valid output code does not guarantee a correct classification.

## Repository files

| File | Purpose |
| --- | --- |
| [README.md](README.md) | Project overview and execution guide |
| [occupation_classification_demo.ipynb](occupation_classification_demo.ipynb) | Self-contained source notebook, ready to run |
| [occupation_classification_demo_executed.ipynb](occupation_classification_demo_executed.ipynb) | Executed notebook with saved tables and outputs; readable without rerunning |
| [Input_Files_Zip](https://github.com/david-shomron/LLM-BATCH-CLASSIFICATION/raw/refs/heads/main/occupation_classification_demo.zip) | `occupation_classification_demo.zip`: the five original input files only |
| [predictions.csv](predictions.csv) | Example final-selector predictions for validation |
| [predictions.xlsx](predictions.xlsx) | The same predictions, plus a **Column descriptions** sheet |

## Input data and model representations

The input ZIP contains **four XLSX files and one DOCX**:

| Source | Role |
| --- | --- |
| Taxonomy hierarchy spreadsheet (`t1`) | Occupation hierarchy and unit titles |
| Occupation registry spreadsheet (`t2`) | Canonical detailed occupations |
| Occupation index spreadsheet (`t3`) | Aliases and role variants |
| Hebrew classification DOCX | Occupation descriptions, duties and exclusions |
| `DUMMY_DATASET.xlsx` | Survey responses and reference occupation labels |

The supplied taxonomy snapshot is `IL_OCC_2026_DRAFT_2026-10-21`, with 7,035 eligible alias rows across 465 four-digit units.

- **Survey `query_text`:** `EzoAvoda, TeurPeula, TeurTafkid, SugAvoda, SugMachlaka, ShemMachlaka`.
- **Taxonomy `text_context`:** alias, occupation/group/unit titles, hierarchy titles and available role title.
- **Occupation cards:** structured taxonomy text with validated DOCX evidence, within an explicit text budget.
- **Reference label:** `SemelMishlachSofi`, used for evaluation rather than model input.

IDs and reference labels are excluded from classification evidence. Query-equivalent cases share inference results, which are then expanded back to the retained validation cases.

## Run the notebook in Google Colab

[![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/david-shomron/LLM-BATCH-CLASSIFICATION/blob/main/occupation_classification_demo.ipynb)

1. Click **Open in Colab**, or open the source notebook from this repository in Colab.
2. Select **Runtime → Change runtime type → GPU** and connect.
3. Select **Runtime → Run all**. Setup installs pinned model libraries using **uv**, preserving Colab’s existing scientific libraries and CUDA-enabled PyTorch. Public model checkpoints download automatically.
4. The **Model Sources** step automatically downloads **Input_Files_Zip** from `main` when the required inputs are missing. No manual file upload is required.
5. Keep the default **K=50, M=10** to run the proposed pipeline. Inspect the stage outputs and evaluation tables as execution progresses.
6. With `EXPORT_PREDICTIONS=True` (the default), download the new `predictions.csv` and `predictions.xlsx` from Colab’s Files panel under `artifacts/demo/`.

Internet access and a CUDA GPU are required for the complete pipeline. No private API, account token or additional project scripts are needed. Setup preserves already-loaded library versions and checks NumPy before importing models. If an earlier setup caused a NumPy mismatch, rerun the updated setup cell and then continue; no session restart is needed for this mismatch.

`INPUT_FILES_ZIP_URL` follows the current `main` branch. Existing local inputs are reused; remove the runtime’s `inputs/` directory before rerunning the source-loading step to fetch the latest published inputs. The repository’s prediction files are examples; they are not loaded as model outputs during execution.

## Evaluation and interpretation

The 200 source records produce **138 retained cases**, with **113 unique usable queries**. Six cases have partial or unknown reference labels. Four cases lack usable survey text and bypass inference; complete-label cases without text remain misses in evaluation.

| Evaluation cohort | Retrieval Recall@50 | Shortlist Recall@10 | Final accuracy | Macro F1 |
| --- | --- | --- | --- | --- |
| 132 complete mapped references | 92.42% | 90.15% | 65.15% | 65.82% |
| 130 after excluding both label-conflict cases | 93.08% | 90.77% | 65.38% | 66.98% |

These are descriptive results on a small dummy dataset, not an estimate of performance on national survey data. Macro F1 averages over reference-supported classes. Validation on representative, adjudicated survey data is required before operational use.

Exports preserve the original `SemelMishlachSofi`; `evaluation_label` shows the mapped reference used in evaluation (`7414→7411`, `5222→5223`). `identical_label` compares the prediction with the original code and is blank for partial/unknown labels, so its average can differ from the reported mapped-label accuracy. The Excel **Column descriptions** sheet explains all exported fields.
