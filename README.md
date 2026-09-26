Business Entity Resolution
A memory-efficient entity resolution pipeline for identifying matching businesses across multiple large-scale business datasets.
📌 Overview
This project addresses the problem of business entity resolution across three different data sources.
Given a business record from Source 1, the objective is to identify all corresponding business entities in Source 2 and Source 3, even when the same business may be represented differently across datasets.
For example:
Source 1
Shree Ganesh Restaurant Pvt Ltd
12 MG Road, Pune

        ↓ Entity Resolution

Source 2
Shree Ganesh Restaurant
12 Mahatma Gandhi Road, Pune

Source 3
Shree Ganesh Restro Pvt. Ltd.
12 MG Rd, Pune

The challenge is complicated by:
Different business-name formats
Address variations
Abbreviations
Spelling variations
Multiple matches for a single Source 1 entity
Different representations across data sources
Extremely large datasets

🎯 Objective
For every Source 1 business, identify all matching entities from Source 2 and Source 3.
The final system is expected to produce:
Source 1 Entity → Matching Source 2 / Source 3 Entities

An important characteristic of this problem is that one Source 1 business can have multiple valid matches, so the pipeline cannot assume a one-to-one relationship.

📊 Dataset Scale
The project works with datasets containing more than 10 million records across Source 2 and Source 3.
Current processed record counts:
Source
Records
Source 2
5,034,616
Source 3
5,285,603

Because of this scale, loading the complete datasets into pandas simultaneously is not practical.

🧠 Memory-Safe Architecture
The initial approach of loading the large datasets directly into pandas caused the SageMaker kernel to run out of memory.
To solve this, the pipeline was redesigned around SQLite-based disk storage and indexing.
Current architecture
Source 2 ──┐
           │
           ├──► SQLite Database
           │        │
Source 3 ──┘        ▼
                Indexed Search
                     │
                     ▼
                 Source 1
                     │
                     ▼
             Candidate Generation
                     │
                     ▼
              Candidate Ranking
                     │
                     ▼
                ML Matching
                     │
                     ▼
              Final Entity Matches

The large datasets are processed in 25,000-row chunks, allowing the system to operate without loading the complete datasets into RAM.

🗄️ SQLite Database
The large Source 2 and Source 3 datasets have been imported into SQLite and indexed.
Completed
Source 2 imported
Source 3 imported
SQLite database created
Required indexes created
Chunk-based processing implemented
Large-scale data remains on disk instead of being loaded entirely into memory
Important
The SQLite database has already been built and should not be unnecessarily rebuilt.

🔎 Candidate Generation
Comparing every Source 1 business against more than 10 million Source 2/Source 3 records would be computationally expensive.
Therefore, the system first performs candidate generation.
Instead of asking:
"Which of the 10+ million records is the match?"
the system first asks:
"Which records are plausible candidates?"
Current blocking strategies include:
1. Country + Name Prefix
Records are grouped using country and the beginning of the normalized business name.
Country + Name Prefix

2. Country + First Name Token
The first meaningful business-name token is used to retrieve potential candidates.
Country + First Name Token

3. Country + Address Tokens
Selected address tokens are used to identify additional candidates.
Country + Address Token

These strategies significantly reduce the search space before similarity matching.

📈 Baseline Candidate Recall
The initial candidate-generation system was evaluated using 100 Source 1 businesses.
There were:
Actual matches = 365
Matches found  = 320
Missed matches = 45

Therefore:
Candidate Recall = 320 / 365

Candidate Recall = 87.67%

Why candidate recall matters
The ML model can only evaluate candidates that reach it.
Therefore, if a genuine match is excluded during candidate generation, the ML model cannot recover it later.
Candidate Generation
        ↓
      87.67%
        ↓
Similarity / ML Model

This makes candidate recall a critical stage of the pipeline.

🔬 Candidate Generation Analysis 
The 45 missed true matches were analyzed individually to determine why they were excluded.
Results
Failure Type
Count
Percentage
Block found but ranked out
37
82.2%
No existing block hit
8
17.8%
Total
45
100%

Key Finding
The majority of missed matches were already retrievable using the existing blocking indexes.
Specifically:
37 / 45 = 82.2%

were found by at least one existing blocking strategy but were not retained in the final candidate set.
Only:
8 / 45 = 17.8%

had no hit in the current blocking strategies.
Conclusion
The primary bottleneck is currently candidate ranking/selection, rather than complete blocking failure.
This finding allowed the next improvement to focus on ranking instead of immediately rebuilding the large SQLite indexing system.

⚙️ Candidate Ranking Improvement 
Based on the missed-match analysis , the candidate-ranking strategy was improved.
Previously, all blocking signals contributed equally:
Name Prefix       +1
First Name Token  +1
Address Token     +1

The improved strategy uses weighted signals:
Blocking Signal
Weight
Name Prefix
+3
First Name Token
+2
Address Token
+1

A multi-signal bonus was also introduced.
Candidates supported by multiple independent signals receive a higher ranking.
Updated ranking logic
Name Prefix
     +3
      │
First Name Token
     +2
      │
Address Token
     +1
      │
Multi-Signal Bonus
      │
      ▼
Weighted Candidate Ranking
      │
      ▼
Top Candidate Set

The purpose is to ensure that candidates with stronger evidence are prioritized before the candidate limit is applied.

🧪 Validation  
The improved candidate-generation strategy is being evaluated using the same 100 Source 1 records and 365 known matches used for the baseline.
Baseline
320 / 365
87.67% recall

Improved system
Validation: In Progress
Matches found: TBD
Candidate recall: TBD

The new result will be compared directly against the 87.67% baseline.
If recall improves sufficiently, the pipeline will proceed toward similarity features and ML matching.
If recall remains insufficient, additional targeted blocking strategies will be investigated.

🤖 Planned ML Matching Stage
Once candidate generation achieves sufficient recall, the pipeline will move to similarity-based matching and machine learning.
For each Source 1–candidate pair, features can include:
Business-name similarity
Address similarity
Country agreement
Token overlap
Character-level similarity
Name length differences
Address length differences
Other structured similarity signals
Example:
S1:
Shree Ganesh Restaurant

Candidate:
Shree Ganesh Restro

Name similarity     → HIGH
Address similarity  → HIGH
Country             → SAME

                    ↓

              ML MATCH SCORE


🧠 Model Training
The final model will not be trained directly on all 10+ million records.
Instead, a manageable training dataset will be constructed containing:
Business A + Business B → MATCH
Business A + Business C → NOT MATCH
Business A + Business D → MATCH

The model will learn the patterns that distinguish genuine entity matches from non-matches.

🎯 Threshold Tuning
The model's decision threshold will be tuned using the competition's evaluation metric.
The competition uses F0.5, which places greater importance on precision than recall.
The threshold will therefore be selected based on validation performance rather than using an arbitrary probability such as 0.5.

🚀 Final Test Pipeline
After candidate generation and model validation are complete, the system will be applied to the test Source 1 dataset.
Current test size:
72,729 Source 1 businesses

The planned pipeline is:
72,729 Test S1 Records
          │
          ▼
   Candidate Generation
          │
          ▼
   Similarity Features
          │
          ▼
      ML Scoring
          │
          ▼
    Threshold Tuning
          │
          ▼
     Final Matches


📄 Expected Output
The final solution is expected to generate:
matching_results.tsv
Example:
source1_entity_id    matched_entity_ids
S1_001               S2_101,S3_205
S1_002
S1_003               S2_301,S2_302,S3_450

An empty matched_entity_ids field indicates that no match was identified.
candidate_pairs.tsv
This file will contain the candidate relationships required by the challenge.
The exact final format will be determined from the official challenge validator/template before submission.

✅ Validation
Before submission, the final output will be checked using the official validation utility:
python3 utils/validate_submission.py \
  --matching output/matching_results.tsv \
  --candidate output/candidate_pairs.tsv \
  --test-dir dataset/test

This will verify the required structure and content of the submission files.

📁 Project Structure
The planned repository structure is:
business-entity-resolution/
│
├── code/
│   └── business_entity_resolution/
│       ├── src/
│       │   ├── memory_safe_pipeline.ipynb
│       │   └── ...
│       ├── README.md
│       └── requirements.txt
│
├── output/
│   ├── matching_results.tsv
│   └── candidate_pairs.tsv
│
├── Documentation_template.md
│
└── .gitignore

Large raw datasets and generated SQLite databases should not be committed to GitHub unless explicitly required.

 📌 Current Project Status

Problem Understanding             ✅
Data Inspection                   ✅
Memory-Safe Architecture          ✅
SQLite Database                   ✅
S2 Processing                     ✅
S3 Processing                     ✅
SQLite Indexing                   ✅
Evaluation Setup                  ✅
Candidate Generation              ✅
Baseline Candidate Recall         ✅ 87.67%
Missed-Match Analysis             ✅
Candidate Ranking Improvement     ✅ Implemented
Candidate Recall Validation       🔄 In Progress
Similarity Features               ⏳
ML Training                       ⏳
Threshold Tuning                  ⏳
Final Test Inference              ⏳
Submission Generation             ⏳
Official Validation               ⏳
Final ZIP                         ⏳



🗺️ Roadmap
Data Understanding
       ↓
Memory-Safe Processing
       ↓
SQLite Database + Indexing
       ↓
Candidate Generation
       ↓
Baseline Recall
       ↓
Missed-Match Analysis
       ↓
Candidate Ranking Improvement
       ↓
Recall Validation
       ↓
Similarity Features
       ↓
ML Model
       ↓
Threshold Tuning
       ↓
Test Data Inference
       ↓
Submission Generation
       ↓
Validation
       ↓
Final Submission



💡 Key Design Principle
The project follows a recall-first candidate generation strategy followed by more precise similarity/ML-based matching.
The system should avoid performing expensive comparisons against the entire 10+ million-record dataset.
Instead:
Efficient blocking reduces the search space, candidate ranking prioritizes strong matches, and machine learning performs the final matching decision.
The architecture is designed to remain memory-safe, scalable, and suitable for large-scale entity resolution.
