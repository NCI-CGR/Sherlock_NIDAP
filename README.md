# **Sherlock Pathology Data Processing Pipeline**



## **1. Overview**



This project implements a comprehensive, rule-based pathology adjudication workflow for NSLC multi-pathologist datasets (P1, P2, P3).

The pipeline:



Cleans and standardizes inputs



Applies pathology-specific logic



Performs agreement checks (histology, subtype, differentiation)



Computes totals, grades, and review flags



Derives case-level Resolution and Status



Produces a final adjudicated dataset with 135 standardized columns



All logic is implemented in Python (Pandas).



Before using the code below please make sure you have the following files that are required as inputs.



1\. Data is extracted from HALO in a wide format in a csv. Raw input from pathologist P1/P2/P3. See template (HALO\_export.csv)

2\. Assigned Pathologist information for each case in csv. See template (allPathassignments07112024.csv)

3\. Fixative information for each case. We added fixative information and case ID on each image as a image level field and exported that information from HALO. See template on how it should the final file look like. (Step3a\_output\_From\_Foundry.csv)





Please note The HALO\_export.csv was exported from NIH Data Integrated Platform(NIDAP). Database connections were created between NIDAP and HALO Database. All the tables from HALO database were imported into NIDAP as foundry objects. Pipelines were created to obtain the data from NIDAP. Data in NIDAP was synced on hourly bases. Having NIDAP is not a prerequisite. Data from HALO can be exported in multiple ways but the final format should match the template(HALO\_export.csv)

1\. Using built in export function in HALO link

2\. Using HALO's API (graphQL)

3\. Using SQL queries from HALO's Database.







## **2. Cleaning \& Standardization**



### 2.1 Normalize missing values



Treat "None", "", whitespace, and NaN as None-like.



Standardize strings using a helper function (s(x)).



### 2.2 Fix histology wording



The following terms are unified across all histology columns:



"Carcinoid, typical" → "Carcinoid"



"Carcinoid, atypical" → "Carcinoid"



"Carcinoid, NOS" → "Carcinoid"



Applied to:



DCEG\_Shrlck\_HistoTypeSingle\_\*



DCEG\_Shrlck\_HistoTypeMixed1\_\*



DCEG\_Shrlck\_HistoTypeMixed2\_\*



Check1\_P\*



### 2.3 Convert subtype percentage fields to integers



Ensures consistent math on subtype totals and grade calculations.



## **3. Check1 Logic (P1, P2, P3)**



Each pathologist gets a Check1\_PX classification following:



Option A



Single-histology consistency rules



Option B



Mixed-histology combination logic



Option C



Interprets “Not Evaluable” cases



NaN-safe string comparisons



All .strip() calls and None/NaN handling prevent errors.



## **4. Adenocarcinoma Filter**

Adeno\_filter



Combines P1, P2, P3 Check1 outputs:



If P3 is None-like → "P1 ; P2"



Otherwise → "P1 ; P2 ; P3"



This drives decision rules for histology and differentiation.



## **5. Primary Histology Comparison**



Outputs:



"all\_agree(2)"



"all\_agree(3)"



"no\_agreement(additional review)"



Rules compare:



Check1 results



P3 involvement



Adenocarcinoma matching



## **6. Subtype Total Computation**

Totals for each pathologist:

getTotal\_subtype\_p1 = lepidic + acinar + pap + mpap + solid + crib

getTotal\_subtype\_p2 = same variables

getTotal\_subtype\_p3 = same variables





Subtype totals must equal 100, otherwise → "Fix subtype".



## **7. High Grade \& Differentiation**

High-grade calculation:

High\_gradePX = mpap + crib + solid



Differentiation:



Rules:



high\_grade >= 20 → "poorly"



If not high-grade:



lepidic ≥ (acinar+pap) → "well"



else → "moderate"



Combined differentiation:



combine\_differentiation = "DiffP1,DiffP2,DiffP3"



Differentiation\_result1



≥2 “well” → well



≥2 “moderate” → moderate



≥2 “poorly” → poorly



≥2 “Not Calculated” → Not Calculated



else → no\_agreement



Hard-coded overrides



Specific case IDs override the computed differentiation value.



## **8. Discussion Flags**



Binary indicators:



Diss\_P1, Diss\_P2, Diss\_P3





NumberofPath = Diss\_P1 + Diss\_P2 + Diss\_P3



## **9. Resolution Engine**



The most complex rule set in the pipeline.

Determines final case adjudication based on:



✔ QC FAIL conditions

✔ P3 None-like vs Non-None

✔ Pending review

✔ Histology agreement \& conflict

✔ Differentiation agreement

✔ Adenocarcinoma matching

✔ Subtype totals

✔ Assign P3 logic

✔ Multi-pathologist agreement

✔ Overrides for specific names



Possible outputs include:



Complete



Assign P3



Fix Histology



Fix subtype



Not Adenocarcinoma



P1/P2 Diff.



P1/P2/P3(atleast2) Diff.



Primary Histology match...



QC FAIL



Pending review(P1/P2/P3)



no category



Discuss (director override)





## **10. Status Classification**



Status is derived from Resolution:





## **11. Final Schema (Step 116)**



The output is column-selected and ordered into a strict schema of 135 fields, matching Spark/Foundry definitions.



df\_final = df\[selected\_columns]





This ensures consistency across environments.





## **12. Sorting Rules**



Sort by "name" ascending



Sort by "Status" ascending



df\_final = df\_final.sort\_values(by=\['Status', 'name'])







## **13. Outputs**

Main deliverables:



df\_final → adjudicated master dataset



Resolution + Status → final case disposition



Differentiation\_result → final grade summary



Primary\_histology\_comparison → agreement logic



QC fields (totals, high-grade, discussion flags)



Optional diff report when comparing against a baseline



Raw data





## **14. Notes**



Pipeline is deterministic; re-running yields identical outputs.



Overrides for specific cases must be updated manually if names change.



Resolution logic heavily depends on correct upstream steps.



P3 logic is sensitive to blank/None distinctions (handled consistently).



All string comparisons normalized via .strip() and lowercase where needed.

