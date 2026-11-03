# **Sherlock Pathology Data Processing Pipeline**

## **1. Overview**

This project implements a Python-based workflow for processing and adjudicating pathology data from multiple pathologists (P1, P2, and P3) in the Sherlock-Lung study.

The pipeline takes pathology review data and case-level information as input, applies predefined quality control (QC) checks and adjudication rules, compares histologic classifications and differentiation grades across reviewers, and generates a standardized case-level dataset with the corresponding adjudication results.

The main processing steps include:

- Cleaning and standardizing input data.
- Evaluating pathology review data and applying QC rules.
- Comparing histologic classifications across pathologists.
- Calculating histologic subtype totals and differentiation grades.
- Identifying cases requiring additional review or discussion.
- Determining the final case-level resolution and status.
- Generating a standardized output dataset for downstream analyses.

All data processing and adjudication rules are implemented in Python using pandas.

### Input Data

Before running the notebook, prepare the following three input CSV files and specify their file paths in the corresponding notebook cells.

**Pathology review data (`HALO_export.csv`)**

Contains the pathology review results exported from HALO in wide format. The dataset includes the histologic assessments, selected histology fields, subtype percentages, differentiation-related information, and QC-related fields for each pathologist (P1, P2, and P3).

The input data must follow the structure provided in the `HALO_export.csv` template.

**Pathologist assignment data (`allPathassignments.csv`)**

Contains the assigned pathologists for each case. The assignment information is used to determine which pathologists are expected to review each case and to apply the corresponding adjudication rules.

**Fixative information (`fixative.csv`)**

Contains image-level tissue fixative information summarized at the case level. The file includes the following fields:

- `g_tissueFixative`: Original fixative information recorded at the image level in HALO.
- `getFixative`: Aggregated fixative information combining all images belonging to the same case.
- `Fixassign`: Simplified case-level fixative classification with three possible categories:
  - `Formalin`: All images were formalin-fixed.
  - `Mixed`: Both formalin-fixed and frozen images were present.
  - `Frozen`: All images were frozen.

The input file must follow the structure provided in the `fixative.csv` template. The notebook uses the supplied case-level fixative fields; it does not calculate them from `g_tissueFixative`.

**Input data preparation:** The pipeline is independent of the data extraction method. Pathology review data and associated case-level information can be obtained through HALO exports, API queries, database queries, or other data extraction methods. Regardless of the extraction method, the resulting CSV files must conform to the corresponding input templates and contain the required fields.

## **2. Cleaning & Standardization**

### 2.1 Normalize missing values

The helper `s(x)` converts missing pandas values to an empty string and strips whitespace in the functions that call it. Some other cells still compare raw values or convert them with `str()`, so blank, `None`.

### 2.2 Fix histology wording

The values `Carcinoid, typical`, `Carcinoid, atypical`, and `Carcinoid, NOS` are replaced with `Carcinoid` in the selected single/mixed histology and `Check1_P1`/`P2`/`P3` columns.

### 2.3 Convert subtype percentage fields to integers

The six subtype percentages per reviewer are converted to numbers; missing or invalid values become zero before totals and grade sums are calculated.

## **3. Check1 Logic (P1, P2, P3)**

Each reviewer receives a `Check1_P1`, `Check1_P2`, or `Check1_P3` classification:

- **Option A:** uses the selected single-histology value and checks it against mixed-histology fields.
- **Option B:** combines the two mixed-histology values, with some consistency checks.
- **Option C:** marks the not-evaluable option `QC FAIL`.
- If P3 has not been assigned, `Check1_P3` is `P3 not assigned`.


## **4. Adenocarcinoma Filter**

Adeno\_filter



Combines P1, P2, P3 Check1 outputs:



If P3 is None-like → "P1 ; P2"



Otherwise → "P1 ; P2 ; P3"



This drives decision rules for histology and differentiation.


## **5. Primary Histology Comparison**

`Primary_histology_comparison` compares `Check1` values and P3 assignment and returns `all_agree(2)`, `all_agree(3)`, or `no_agreement(additional review)`. `header_comparison_test` separately compares the reviewers' single-versus-mixed selections. Both fields feed the Resolution rules.

## **6. Subtype Total Computation**

For each reviewer:

`getTotal_subtype_pX = lepidic + acinar + papillary + micropapillary + solid + cribriform`

Subtype totals must equal 100, otherwise → "Fix subtype".


## **7. High Grade & Differentiation**

High-grade sum for each reviewer:

`High_gradePX = micropapillary + cribriform + solid`

The reviewer differentiation rules are:

- High-grade sum ≥ 20: `poorly`.
- High-grade sum < 20 and lepidic > max(acinar, papillary): `well`.
- High-grade sum < 20 and lepidic < max(acinar, papillary): `moderate`.
- High-grade sum < 20 and lepidic = max(acinar, papillary): `None` (exported as blank).

The calculated grade is retained only when the reviewer's `Check1` is exactly `Adenocarcinoma, non-mucinous`; otherwise the notebook writes `Not calculated1`, `Not calculated2`, or `Not calculated3`.

`combine_differentiation` joins the three reviewer values. `Differentiation_result1` returns a grade seen at least twice (`well`, `moderate`, or `poorly`); when at least two values say `Not calculated`, it returns `Not Calculated`; otherwise it returns `no_agreement`. Two later case-ID mapping steps can replace `Differentiation_result`.

`Compare_differentiation` compares reviewer values after converting them to strings.

## **8. Discussion Flags**

`Diss_P1`, `Diss_P2`, and `Diss_P3` are 1 only when the corresponding `DCEG_Shrlck_add_discussion_PX` field is `Yes`, ignoring case and spaces. A blank or `No` produces 0.

`NumberofPath = Diss_P1 + Diss_P2 + Diss_P3` counts **discussion flags**, not the number of assigned pathologists. After the other calculated Resolution checks, a `Yes` flag from any reviewer changes the calculated Resolution to `Discuss`.

## **9. Resolution Engine**

The notebook calculates Resolution from `Check1` QC failures, reviewer completion, P3 assignment, histology and differentiation agreement, subtype totals, and adenocarcinoma classification. Possible calculated values include `Complete`, `Assign P3`, `Fix Histology`, `Fix subtype`, `Not Adenocarcinoma`, `P1/P2 Diff.`, `P1/P2/P3 Diff.`, `No agreement`, `Primary Histology match; diff. disagreement`, `QC FAIL`, `Pending review(...)`, and `Discuss`.



## **10. Status Classification**

Status is derived from Resolution


## **11. Final Schema**

The notebook defines a list of **123 selected columns** in a fixed order. It builds `df_final` using only selected columns actually present in `df`, silently omitting a selected field if it is missing.

The last formatting cell copies `df_final` and adds an `exclude` field, so the written file has **up to 124 columns** (123 selected plus `exclude`). The extra field does not remove rows.

## **12. Sorting Rules**

The final dataset is sorted by name(CaseID) and then Status, both ascending, before export.”

## **13. Outputs**

The notebook writes the following CSV "Pythonoutput.csv" in its configured working directory.

Main deliverables:

df_final → adjudicated master dataset

Resolution + Status → final case disposition

Differentiation_result → final grade summary

Primary_histology_comparison → agreement logic

QC fields (totals, high-grade, discussion flags)

Optional diff report when comparing against a baseline

Raw data

## **14. Notes**

Pipeline is deterministic; re-running yields identical outputs.

Overrides for specific cases must be updated manually if names change.

Resolution logic heavily depends on correct upstream steps.

P3 logic is sensitive to blank/None distinctions (handled consistently).

All string comparisons normalized via .strip() and lowercase where needed.
