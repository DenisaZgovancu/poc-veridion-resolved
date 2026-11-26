# Veridion POC Resolution

### 1 Data understanding
- Initial file has 2,951 rows --> **592 distinct client inputs** (1 input to many candidates, up to 5 each).
- No fully duplicated rows. `input_row_key` is numeric only (in PowerQuery during EDA there seems to be a problem, most likely from `ILLEGAL_CHARACTERS` - eliminated for output).
- I split the file into an **input table** (columns `input_`) and an **output table** (candidate companies with matching columns).
- Most of this was initially made in Power Query. I used PowerBI since I have an old Excel with limited capabilities (e.g does not have fuzzy match)

### 2 Input duplicates
Two inputs can describe the same company (e.g. `APPLE DISTRIBUTION INTERNATIONAL LIMITED`, rows 44 and 45, identical except for nulls). I did not drop them, because a null in one row does not prove it equals the value in the other.
After cleaning company names I observed multiple in this situation. I did not keep the flag in the output, but I could raise this discussion and see if is worth changing something or leave everything as it is.
In the original notebook I have a column for this that could be exported if needed.

### 3 Cleaning
- Lowercase, strip accents, remove punctuation.
- Company names: remove legal-form words (ltd, gmbh, sa, etc.); other fields: remove generic place words (region, city, county...).
- Name columns of the output (`company_name`, legal names, commercial names) are unpivoted and split on `|`, so an input name can be compared with **any** known name of a candidate.
- The `locations` column is split into one row per location, so the client's address can match **any** known address of a candidate, not only the main one.
- The company's own country name is removed from the names before comparing, so a shared country word does not inflate similarity.

### 4 Matching logic
For each input row, candidates are restricted to the **same country code** and **input_row_key**, then scored:

- **Name score:** Jaro-Winkler similarity between cleaned names (best score across all known names of the candidate).
- **Address score:** share of region, city, postcode, street and street number that match exactly (best across known locations).
- **Final score** = 0.6 x name score + 0.4 x address score. If the name score is 70 or lower, the final score is 0 - I observed many artificially mathed bu same location, so I increased name score weight and decided on a limit.
- **Confidence category:** Exact (final score above 99), High (final score above 85 or name score above 85), Medium (final score above 50 or name score above 70), Low (everything else).
- The top-ranked candidate per input is kept (only first value - worth checking in future if we should add a gap column to measure the number 1 vs number 2 distance in score).

### 3.6 Duplicates in the client's supplier list
The combination row_key + veridion ids is unique, but we have 18 duplicate veridion ids. I kept them in the file, but we should check if the input is the same company duplicated in the input, or some are not a real match.

## 4. Data QC

Flags are added in Power Query for the:
- Different Country: this also means those were matched using locations 
- year < 1800 or year > 2026 - one example Posten Bring (correct example, I already verified)
- Low score or none (not a match) - I deleted the match
- Duplicated veridion ids (Pivot) 

### Proposed curation plan
- **Fix or drop:** Low scores
- **Flag for the client:** Shared matches, so category managers can confirm them and medium match since we are not sure
- **Ask the client:** whether duplicate groups should be merged into one supplier record, and which address to treat as primary. I will also propose a discussion about the main in the input vs main in the output
- **Refresh:** records not updated in over 12 months should be re-verified

## 5. Limitations

I kept the matching simple for a POC. Known weaknesses:

- **Missing values count as mismatches** 
- **Weights and thresholds** (0.6/0.4, 70, 85) are judgment calls - on a high scale I would investigate **Splink**
- **Ties using row number** are broken arbitrarily by the ranking
- **A name-only match can reach "High"** even with weak address agreement, which is risky for chains and common names.
- **Splink** was explored for profiling only. The matching itself is custom SQL in DuckDB.
- **Still some unsure rows** the medium ones could benefit from manually tracking while the low and unmatched ones need work finding a coresponding company online

