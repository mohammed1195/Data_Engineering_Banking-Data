# Identifying Bad Data

## Purpose

This notebook flags member IDs that appear more than once in three LendingClub source tables. The team treats all records for a flagged member ID as ambiguous, combines the flagged IDs into one exclusion list, and uses that list to produce the cleaned_new tables.

The notebook output shows conflicting values for repeated IDs, including different total_high_credit_limit values, delinquency history, and recent inquiry counts. In the customer example, the notebook comments that the value could not be selected safely and that the issue was escalated to stakeholders.

## Workflow

~~~mermaid
flowchart LR
    C["customers<br/>3,157 duplicate IDs"] --> U["Union ID lists"]
    D["loans_defaulters_delinq<br/>173 duplicate IDs"] --> U
    R["loans_defaulters_detail_rec_enq<br/>3,189 duplicate IDs"] --> U
    U --> X["Distinct combined list<br/>3,189 member IDs"]
    X --> F["Exclude flagged IDs<br/>from selected source tables"]
    F --> N["Write cleaned_new Parquet<br/>and register Hive tables"]
~~~

## Findings from saved notebook outputs

![Counts of duplicate member IDs by source and in the combined exclusion set](./identifying_bad_data_duplicate_member_ids.svg)

These are counts of distinct member_id groups whose row count is greater than one; they are not counts of excess rows.

- **Customers:** 3,157 duplicate member IDs.
- **Delinquency:** 173 duplicate member IDs.
- **Detailed records and enquiries:** 3,189 duplicate member IDs.
- **Combined exclusion set:** 3,189 distinct member IDs.
- **Cleaned customer table:** 2,254,192 rows after exclusion.

The combined set has the same count as the detailed-records/enquiries set. Given the notebook's grouped ID lists and distinct union, that means the other two sets contributed no additional IDs in the saved run. Confirm this with explicit set-difference checks before relying on the overlap as a permanent data property.

The notebook's worked example shows:

- Five customer rows for one member, with total_high_credit_limit changing between rows.
- Three delinquency rows for that member, with delinq_2yrs values of 3, 1, and 1 and mths_since_last_delinq values of 0, 0, and 45.
- Five detail/enquiry rows, with inq_last_6mths values of 0, 3, 2, 1, and 0.

Those differences support treating the examples as unresolved conflicts under a one-row-per-member assumption. They do not, by themselves, prove that every repeated member ID in every source is invalid; confirm the intended grain and whether snapshots or loan-level records are expected.

## What the notebook writes

The combined bad-member ID list is saved under the project's bad output. The notebook then filters flagged IDs from the customer, delinquency, and detailed-record/enquiry tables, writes the resulting Parquet data under cleaned_new, and registers external Hive tables for those outputs.

The saved notebook output includes a post-filter customer count, but it does not show post-filter row counts for the other two tables. It also does not show source-specific exports for all three duplicate lists, so verify the actual folder contents if those separate files are part of the handoff.

## Analyst recommendations

1. **Confirm the key grain.** If the customer table is intended to contain one row per member, validate that assumption with the data owner. Keep a separate rule for legitimate repeated snapshots or loan-level rows.
2. **Separate exact duplicates from conflicting duplicates.** Preserve the source rows, a reason code, and the detected values so an analyst can distinguish identical repeats from material conflicts.
3. **Check set overlap explicitly.** Compare each source ID set with the combined set and report intersections and source-only IDs.
4. **Check missing-ID handling.** The most frequent detailed-record/enquiry ID in the saved output begins with e3b0c44298fc1c149…, which is consistent with a SHA-256 hash of an empty string. If IDs are hashed upstream, confirm whether this represents a blank source ID and handle it as missing rather than as an ordinary member.
5. **Guard against nulls in exclusions.** The notebook uses NOT IN. If the bad-ID list ever contains a true SQL NULL, that predicate can filter out more rows than intended. A left-anti join or a NOT EXISTS rule with explicit null handling is safer.
6. **Reconcile all output counts.** Record input rows, flagged IDs, excluded rows, retained rows, and post-write counts for every table.

## Scope and reproducibility

This README documents the code and saved outputs embedded in Identifying_Bad_Data.ipynb; the Spark/YARN notebook was not rerun. Counts can change when the source tables or cluster data change. The notebook's HDFS locations are environment-specific.
