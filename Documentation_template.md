Business Entity Resolution — Methodology Documentation

1. Problem Statement

The objective of this project is to perform large-scale Business Entity Resolution (BER) across three independent business datasets.

Source 1 is the deduplicated reference source. For every Source 1 entity, the system identifies all corresponding entities from Source 2 and Source 3.

A Source 1 entity may have no match, one match, or multiple matches.

The main challenge is that the sources do not share a common identifier across datasets. Business names and addresses can also differ because of abbreviations, spelling variations, formatting differences, legal suffixes, transliterations, and incomplete address information.

The solution therefore combines scalable candidate generation with machine-learning-based matching.

2. Dataset Description

Each business record contains:

entity_id

business_name

business_address

country

The entity ID prefix identifies the source:

S1- → Source 1

S2- → Source 2

S3- → Source 3

The processed reference datasets contain:

Source

Records

Source 2

5,034,616

Source 3

5,285,603

The test Source 1 dataset contains:

Dataset

Records

Test Source 1

1,732,544

The scale of Source 2 and Source 3 makes exhaustive pairwise comparison impractical.

3. Constraints and Design Considerations

An initial approach that loaded the large Source 2 and Source 3 datasets directly into pandas caused memory limitations in the SageMaker environment.

The pipeline was redesigned around:

Chunk-based data processing

SQLite disk storage

Database indexing

Candidate generation before expensive feature computation

Checkpointed inference

Parallel processing for large-scale test inference

The large reference datasets remain on disk and only required candidate records are retrieved for each Source 1 entity.

No external business databases, APIs, geocoding services, or external entity-resolution services are used. Matching relies on the provided challenge data and the locally trained model.

4. Overall Methodology

Source 2 + Source 3
        |
        v
SQLite Database
        |
        v
Indexed Candidate Generation
        |
        v
Source 1 Entity
        |
        v
Candidate Retrieval
        |
        v
Similarity Feature Engineering
        |
        v
Random Forest Classifier
        |
        v
Match Probability
        |
        v
Probability Threshold
        |
        v
Final Entity Matches
        |
        +--------------------+
        |                    |
        v                    v
candidate_pairs.tsv   matching_results.tsv

The pipeline consists of:

Data ingestion and memory-safe storage

Candidate generation

Candidate recall evaluation

Missed-match analysis

Feature engineering

Random Forest training

Threshold selection

Large-scale test inference

Submission generation

Official validation and sanity checks

5. Data Ingestion and Memory-Safe Processing

Source 2 and Source 3 are processed in chunks rather than loaded completely into memory.

The data is imported into SQLite in batches of approximately 25,000 records.

SQLite provides persistent on-disk storage and allows candidate records to be retrieved without keeping the complete reference datasets in RAM.

The database stores the fields required for retrieval and feature generation, including:

entity_id

business_name

business_address

country

Indexes are created to make candidate lookup efficient.

Once created, the SQLite database can be reused for subsequent candidate-generation and inference stages.

6. Candidate Generation / Blocking Strategy

6.1 Motivation

A Source 1 entity cannot realistically be compared with every record among the more than 10 million Source 2 and Source 3 records.

Candidate generation therefore reduces the search space before similarity calculation and machine-learning inference.

6.2 Candidate Signals

The implemented candidate-generation process uses signals including:

Country matching

Business-name prefix matching

First-token matching

Address-token overlap

Multiple blocking conditions

Candidate ranking

Candidate IDs are restricted to Source 2 and Source 3 entities.

6.3 Candidate Retrieval

For each Source 1 entity:

Candidate-generation rules are applied.

A bounded candidate set is retrieved.

Candidate IDs are filtered to retain only S2- and S3- entities.

Corresponding records are retrieved from SQLite.

Similarity features are generated.

Candidates are passed to the trained Random Forest model.

This prevents the model from evaluating millions of irrelevant records for every Source 1 entity.

7. Candidate Recall Evaluation

Candidate generation was evaluated against known matching relationships in the training data.

The evaluation identified:

365 actual matching relationships

320 successfully retrieved relationships

45 initially missed relationships

The resulting baseline candidate recall was:

87.67%

Candidate recall is critical because a true match that is removed during blocking cannot be recovered by the downstream classifier.

8. Missed-Match Analysis

The 45 initially missed relationships were analyzed to understand weaknesses in candidate generation.

Reason

Count

Block found but candidate ranked out

37

No existing block hit

8

Total

45

The analysis showed that misses occurred both because a genuine candidate did not satisfy an existing blocking condition and because a generated candidate did not survive candidate ranking.

This analysis informed improvements to candidate generation and ranking.

9. Feature Engineering

For each Source 1/candidate pair, the pipeline generates similarity features.

The Random Forest model uses:

Feature

Description

name_similarity

Similarity between business names

address_similarity

Similarity between business addresses

name_token_overlap

Token-level overlap between business names

address_token_overlap

Token-level overlap between addresses

name_exact

Exact name-match indicator

address_exact

Exact address-match indicator

country_match

Country-match indicator

name_prefix_match

Business-name prefix-match indicator

first_token_match

First-token-match indicator

These features convert each candidate pair into a numerical representation suitable for classification.

10. Machine Learning Model

The matching stage uses a Random Forest classifier.

For each candidate pair, the model receives the engineered feature vector and produces class probabilities.

The positive-class probability is treated as the model's match confidence.

The trained model is stored at:

code/business_entity_resolution/src/analysis/random_forest_entity_matching_model.pkl

The selected matching threshold is stored at:

code/business_entity_resolution/src/analysis/best_matching_threshold.txt

Keeping the threshold separate from the model makes the inference decision boundary reproducible without retraining.

11. Threshold-Based Matching

For every candidate pair, the Random Forest produces a probability for the positive matching class.

A candidate is retained when:

prediction_probability >= selected_matching_threshold

The inference pipeline reads the selected threshold from best_matching_threshold.txt.

Candidates below the threshold are excluded from the final matching results.

12. Large-Scale Test Inference

The test Source 1 dataset contains 1,732,544 records.

Inference is performed using checkpointed, parallel processing.

The implementation uses:

Multiple worker processes

Read-only SQLite connections per worker

Bounded candidate retrieval

Batch processing

Intermediate worker output files

Checkpoints

Final result merging

Each worker:

Opens a read-only SQLite connection.

Loads the trained Random Forest model.

Retrieves candidates for each Source 1 entity.

Retrieves candidate records from SQLite.

Computes similarity features.

Generates Random Forest probabilities.

Applies the selected threshold.

Writes qualifying matches to an intermediate TSV.

Worker outputs are then merged. Duplicate (source1_entity_id, candidate_entity_id) relationships are removed before the final inference output is created.

The intermediate merged inference output is:

output/inference_results_temp.tsv

13. Final Matching Results

The final output is:

output/matching_results.tsv

It contains exactly one row for every Source 1 test entity.

Required columns:

source1_entity_id
matched_entity_ids

matched_entity_ids contains a comma-separated list of matching Source 2 and Source 3 entity IDs.

Example:

source1_entity_id    matched_entity_ids
S1-00001             S2-00047,S2-00193,S3-00812
S1-00002             S3-00004
S1-00003

An empty value means that no sufficiently confident match was identified.

The pipeline does not force an uncertain match merely to avoid an empty result.

14. Candidate Pairs Output

The final candidate set is stored in:

output/candidate_pairs.tsv

This represents the candidate set immediately before the final machine-learning inference stage.

Each Source 1 entity has one row with its candidate Source 2 and Source 3 IDs.

Example:

source1_entity_id    candidate_entity_ids
S1-00001             S2-00047,S2-00193,S3-00812
S1-00002             S3-00004
S1-00003

Every entity included in matching_results.tsv should also appear in the candidate list for that Source 1 entity.

15. Official Validation

The final files are checked using the challenge's official validator:

python3 utils/validate_submission.py     --matching output/matching_results.tsv     --candidate output/candidate_pairs.tsv     --test-dir dataset/test

The validator checks the required submission structure and the consistency of the two output files.

The validator is run before final submission.

16. Output Quality Checks

Additional sanity checks are performed for:

Correct Source 1 coverage

The final matching file must contain exactly one row for every Source 1 test entity.

Expected count:

1,732,544

Duplicate Source 1 records

Every source1_entity_id must occur exactly once.

Valid matched IDs

Every ID in matched_entity_ids must:

Start with S2- or S3-

Exist in the test reference data

Not be an S1 ID

No S1 → S1 matches

Source 1 entities must never be matched to other Source 1 entities.

Duplicate IDs within a match list

The same entity ID must not appear more than once for a Source 1 entity.

Empty results

Empty matching lists are valid when no sufficiently confident match is identified.

Candidate consistency

Every final matched entity must be present in the candidate set for that Source 1 entity.

TSV format

Both output files must use tab-separated values and the exact required column names.

17. Evaluation Considerations

The challenge evaluates matching results using F_0.5, which places greater weight on precision than recall.

The formula is:

F_0.5 = (1.25 × Precision × Recall)
        / (0.25 × Precision + Recall)

The pipeline therefore uses candidate blocking followed by Random Forest probability thresholding rather than accepting every retrieved candidate as a match.

Singleton Source 1 entities are preserved. If no sufficiently confident match is found, their matching list remains empty.

18. Computational Efficiency

The main techniques used to scale the solution are:

SQLite-based storage

Large reference datasets remain on disk instead of being loaded entirely into RAM.

Chunk processing

Large datasets are processed in manageable batches.

Indexed retrieval

SQLite indexes reduce unnecessary database scanning.

Candidate blocking

Only plausible candidates proceed to feature generation and ML inference.

Bounded candidate retrieval

Candidate generation limits the number of candidates evaluated for each Source 1 entity.

Parallel inference

Test inference is distributed across worker processes.

Checkpointing

Intermediate results are written to disk during long-running inference.

19. Reproducibility

The final project package contains:

business-entity-resolution/
│
├── output/
│   ├── matching_results.tsv
│   └── candidate_pairs.tsv
│
├── code/
│   └── business_entity_resolution/
│       ├── src/
│       ├── README.md
│       └── requirements.txt
│
└── Documentation_template.md

The code directory contains the pipeline implementation and model artifacts required by the project.

The methodology document describes the candidate-generation strategy, feature engineering, machine-learning model, inference process, and validation procedure.

20. Limitations

Candidate-generation ceiling

A genuine match that is not retrieved during candidate generation cannot be recovered by the downstream model.

The measured baseline candidate recall of 87.67% demonstrates the importance of continued attention to blocking quality.

No external data augmentation

The system does not use external business databases, APIs, geocoding services, or internet-based business lookup. Matching is performed using the provided challenge data.

Noisy business information

Business names and addresses can be incomplete, abbreviated, misspelled, transliterated, or formatted differently.

Large-scale computation

Processing more than ten million reference records and more than 1.7 million test Source 1 records requires disk-based storage, batching, checkpointing, and parallel inference.

21. Summary

The implemented solution combines scalable candidate generation with machine-learning-based entity matching.

The core workflow is:

Large Reference Datasets
        ↓
Memory-Safe SQLite Storage
        ↓
Indexed Candidate Generation
        ↓
Candidate Retrieval
        ↓
Similarity Feature Engineering
        ↓
Random Forest Classification
        ↓
Probability Thresholding
        ↓
Final Entity Matches

The system produces:

output/candidate_pairs.tsv
output/matching_results.tsv
