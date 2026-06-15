# What I did, how I did it, and why?

## Overview of the methodology

The objective here is to take two datasets, one from the hospital side and another from the payer side, align the records somehow, calculate the difference in rates and assess how much we can trust each comparison.
Here is an overview of how I approached this problem:

- Identify a set of columns in each dataset that we can use to definitively link the two. For example, if two rows from the datasets don't have the same hospital, it's clearly wrong to compare them to one another. I will call these the "join columns".
- Link the two datasets using "join columns" but do a full outer join so we keep records that do not match the other dataset. This allows us to calculate a new field "match_source" that can be either of the datasets or both.
- Identify information in one dataset or both that we can use to assess whether two rows from the datasets lend themselves reasonably to comparison. I will call these the "evaluation columns". These are different than the "join columns" because there is some fuzziness around these. For example, the hospital dataset has a column "setting" which contains "inpatient", "outpatient" or "both". If we could determine some sort of setting from the payer dataset, then we could use these two columns to assess the chances that a match might involve the same setting.
- Devise a measure of "confidence" in the match. The higher the confidence, the more likely two rows might correspond to the "same thing".
- Combine "confidence" and "match_source" into a new field that puts each match into a tier, from highly reliable to less reliable.

## Details of the methodology

### Determining join columns
Obvious fields here were the hospital name, payer name, code type and code. There is also a column in the hospital dataset named "standard_charge_methodology" and a column in the payer dataset named "negotiation_type" that seemed to be highly related to another both semantically and in terms of the types of values they include. I will go through my thoughts on each and details on what I did there.

- **Hospital name:** there is no such field in the payer dataset, but there is EIN which is Employer ID Number from IRS. I noticed that the hospital dataset has a column with source file, which does have the EINs for each hospital. I decided to build a map based on this and use that to populate the name field in the payer dataset.
    - For scaling purposes, it would make sense to use a public API like one Propublica has. We could also populate a lookup table as we ingest files, scraping the EIN from the file, only falling back to the API for file names that do not include the EIN.
- **Payer name:** some normalization was needed here because payer names were coming in lower and upper case as well as abbreviations. I built a map to normalize this to what's there in the payer dataset. We could scale this up by having a map of common abbreviations but could also use similarity measures like Levenshtein distance to capture minor and uncommon misspellings (e.g. United Healthcare vs Unided Healthcare).
- **Code type:** I noticed that there is a code type called "LOCAL" in hospital dataset. I am not sure what that was about but it only came from NYU Langone so I thought maybe that's something weird for that hospital. Those codes looked like CPT codes too, so I decided to change that to CPT.
    - This might be questionable: what if this is how some hospitals bill different types of the same procedure, for some reason? I would want to see if it pops up in any other hospital, and maybe also ask some SMEs in hospitals where it does.
- **Code:** I saw that in some instances, the code type was included in the code, e.g. "MS-DRG 872", so I separated code type from code for those cases.
- **Charge methodology:** I noticed that there are two differently-named columns in the two datasets that seemed to include very similar/same charging and negotiation types, like "percentage" and "fee schedule". I defined a map to normalize these in the two datasets. I noticed a couple things with the hospital dataset. First, there was "other" which is not clearly associated with any negotiation type, so I left it as is rather than assuming anything. Second, there were a small number of rows where the methodology was some numeric value. I checked these rows all of which came from NYU Langone. It looked like there might have been some column shift in those, presumably during export. I could not tell where exactly this shift might have occurred so I did not make any assumptions and left it as is. These rows will not match anything in the payer dataset but there are so few of them, I am fine with it for now. It would be good to have some type checks during ingest so we can mark those erroneous rows automatically and monitor it.

### Evaluation columns
These columns are not used in left join, but to calculate a measure of "confidence" that two rows that are matched on join columns might be talking about the "same thing".