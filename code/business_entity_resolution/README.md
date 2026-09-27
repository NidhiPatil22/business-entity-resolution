# Business Entity Resolution

A memory-efficient machine learning pipeline for identifying matching business entities across multiple large-scale business datasets.

## 📌 Overview

This project focuses on **Business Entity Resolution (BER)** across three different data sources.

Given a business record from **Source 1**, the goal is to identify corresponding business entities in **Source 2** and **Source 3**, even when the same real-world business is represented differently across datasets.

For example:

### Source 1

> Shree Ganesh Restaurant Pvt Ltd
> 12 MG Road, Pune

### Source 2

> Shree Ganesh Restaurant
> 12 Mahatma Gandhi Road, Pune

### Source 3

> Shree Ganesh Restro Pvt. Ltd.
> 12 MG Rd, Pune

Although the records are represented differently, they may refer to the same real-world business.

---

## 🎯 Objective

For every Source 1 business, identify all corresponding entities from Source 2 and Source 3.

The final system produces relationships of the form:

```text
Source 1 Entity → Matching Source 2 / Source 3 Entities
```

An important characteristic of this problem is that a single Source 1 business can have **multiple valid matches**. Therefore, the pipeline does not assume a one-to-one relationship.

---

## 🧩 Challenges

The datasets contain several variations that make direct matching difficult:

* Different business-name formats
* Address variations
* Abbreviations
* Spelling variations
* Different representations across data sources
* Multiple matches for a single Source 1 entity
* More than 10 million records across Source 2 and Source 3
* Memory limitations during large-scale processing

A direct comparison of every Source 1 entity against every Source 2 and Source 3 entity would be computationally expensive.

---

## 📊 Dataset Scale

The processed datasets contain:

| Source   |   Records |
| -------- | --------: |
| Source 2 | 5,034,616 |
| Source 3 | 5,285,603 |

This gives more than **10 million records** across the two reference sources.

The test dataset contains:

| Dataset       |   Records |
| ------------- | --------: |
| Source 1 Test | 1,732,544 |

Loading all datasets into pandas simultaneously is not practical in a memory-constrained environment.

---

# 🧠 Memory-Safe Architecture

The initial approach of loading the large datasets directly into pandas caused memory limitations in the SageMaker environment.

The pipeline was therefore redesigned around **SQLite-based disk storage and indexed candidate retrieval**.

### Pipeline

```text
Source 2 + Source 3
        ↓
   SQLite Database
        ↓
 Indexed Candidate Search
        ↓
      Source 1
        ↓
 Candidate Generation
        ↓
 Candidate Retrieval
        ↓
 Similarity Feature Extraction
        ↓
 Random Forest Model
        ↓
 Probability Threshold
        ↓
 Final Entity Matches
        ↓
 Submission Files
```

The large datasets are processed in chunks so that the complete reference datasets do not need to reside in RAM simultaneously.

---

# 🗄️ SQLite Database

Source 2 and Source 3 are stored in a SQLite database to support memory-efficient candidate retrieval.

### Implemented

* Source 2 imported into SQLite
* Source 3 imported into SQLite
* SQLite database created
* Required indexes created
* Chunk-based processing implemented
* Candidate records retrieved from disk when required
* Large reference datasets kept outside the main pandas memory footprint

The SQLite database can be reused for subsequent inference instead of rebuilding it unnecessarily.

---

# 🔎 Candidate Generation

Comparing every Source 1 business against more than 10 million reference records would be computationally expensive.

The pipeline therefore uses a **candidate-generation stage** before machine-learning prediction.

Instead of comparing a Source 1 entity against every record, the system first identifies a smaller set of plausible candidates.

Candidate-generation signals include:

* Country matching
* Business-name prefix matching
* First-token matching
* Address-token overlap
* Multiple blocking conditions
* Candidate ranking

The resulting candidates are then passed to the feature-engineering and machine-learning stages.

---

# 📈 Candidate Recall Evaluation

Candidate generation was evaluated using a subset of Source 1 records.

The evaluation identified:

* **365 actual matching relationships**
* **320 matches successfully retrieved**
* **45 matches initially missed**

This resulted in a baseline candidate recall of approximately:

**87.67%**

Candidate recall is important because the machine-learning model can only evaluate candidates that are successfully retrieved during candidate generation.

If a genuine match is not present in the candidate set, the downstream model cannot recover it.

---

# 🔍 Missed-Match Analysis

The initially missed relationships were analyzed to understand weaknesses in the candidate-generation process.

The analysis identified two main categories:

| Reason                               |  Count |
| ------------------------------------ | -----: |
| Block found but candidate ranked out |     37 |
| No existing block hit                |      8 |
| **Total**                            | **45** |

This analysis was used to improve the candidate-generation and ranking strategy while keeping the search computationally manageable.

---

# 🏗️ Feature Engineering

After candidate generation, similarity features are calculated between the Source 1 business and each candidate entity.

The current Random Forest matching model uses the following features:

| Feature                 | Description                           |
| ----------------------- | ------------------------------------- |
| `name_similarity`       | Similarity between business names     |
| `address_similarity`    | Similarity between business addresses |
| `name_token_overlap`    | Token-level overlap between names     |
| `address_token_overlap` | Token-level overlap between addresses |
| `name_exact`            | Exact business-name match indicator   |
| `address_exact`         | Exact address match indicator         |
| `country_match`         | Country match indicator               |
| `name_prefix_match`     | Business-name prefix match indicator  |
| `first_token_match`     | First-token match indicator           |

These features are used by the machine-learning model to estimate the probability that a candidate represents the same real-world business.

---

# 🤖 Machine Learning Model

The entity matching stage uses a **Random Forest classifier**.

For every generated candidate pair, the pipeline:

1. Retrieves the candidate entity from SQLite.
2. Generates similarity features.
3. Passes the features to the trained Random Forest model.
4. Obtains a match probability.
5. Applies the selected probability threshold.
6. Keeps candidates whose predicted probability meets the threshold.

Conceptually:

```text
Source 1 Entity
      +
Candidate Entity
      ↓
Similarity Features
      ↓
Random Forest
      ↓
Match Probability
      ↓
Threshold
      ↓
Match / No Match
```

The trained model and threshold are stored under the project code/analysis directory.

---

# 🧪 Test Inference

The trained model is applied to the complete Source 1 test dataset.

The test dataset contains:

```text
1,732,544 Source 1 records
```

The inference pipeline uses checkpointed, parallel processing to make large-scale inference more manageable.

The pipeline:

* Processes Source 1 in batches
* Generates candidates using the SQLite database
* Computes similarity features
* Applies the trained Random Forest model
* Filters candidates using the selected threshold
* Stores intermediate results
* Merges worker outputs
* Removes duplicate Source 1 → candidate relationships
* Produces the final inference output

The intermediate inference output is stored as:

```text
output/inference_results_temp.tsv
```

---

# 📄 Final Submission Files

The final pipeline produces two important files:

```text
output/
├── candidate_pairs.tsv
└── matching_results.tsv
```

## `candidate_pairs.tsv`

Contains the candidate relationships considered during the entity-resolution process.

Conceptually:

```text
Source 1 Entity → Candidate Source 2 / Source 3 Entity
```

This file represents the candidate search space used by the matching pipeline.

---

## `matching_results.tsv`

Contains the final predicted matches.

Each Source 1 entity has a corresponding result containing its matched Source 2 and/or Source 3 entity IDs.

A Source 1 entity may have multiple matches.

If no sufficiently confident match is found, the result may be empty.

Conceptually:

```text
S1-001    S2-100,S3-500
S1-002    S2-200
S1-003
```

An empty result indicates that no candidate passed the matching criteria.

---

# ✅ Official Validation

Before finalizing the outputs, the submission files are checked using the project's official validator:

```bash
python3 utils/validate_submission.py \
    --matching output/matching_results.tsv \
    --candidate output/candidate_pairs.tsv \
    --test-dir dataset/test
```

The validator is used to identify formatting and submission-structure problems before the final output is submitted.

---

# 🔎 Output Quality Checks

In addition to the official validator, the final output is checked for:

* Correct number of Source 1 records
* Duplicate Source 1 records
* Valid Source 2 and Source 3 entity IDs
* Accidental Source 1 → Source 1 matches
* Empty matching results
* Correct TSV formatting
* Consistency between candidate pairs and final matches

The test output is expected to contain one result for each of the **1,732,544 Source 1 test records**.

---

# 📁 Project Structure

```text
business-entity-resolution/
│
├── code/
│   └── business_entity_resolution/
│       │
│       └── src/
│           ├── analysis/
│           │   ├── random_forest_entity_matching_model.pkl
│           │   └── best_matching_threshold.txt
│           │
│           └── ...
│
├── output/
│   ├── candidate_pairs.tsv
│   └── matching_results.tsv
│
├── Documentation_template.md
├── README.md
├── requirements.txt
└── .gitignore
```

Large raw datasets and temporary/generated files that are not required for reproducibility should not be committed to the repository.

---

# ⚙️ Reproducibility

The general workflow for reproducing the pipeline is:

```text
1. Load Source 2 and Source 3
          ↓
2. Build SQLite database
          ↓
3. Create indexes
          ↓
4. Generate Source 1 candidates
          ↓
5. Evaluate candidate recall
          ↓
6. Analyze missed matches
          ↓
7. Generate similarity features
          ↓
8. Train Random Forest
          ↓
9. Select matching threshold
          ↓
10. Run test inference
          ↓
11. Generate candidate_pairs.tsv
          ↓
12. Generate matching_results.tsv
          ↓
13. Run official validator
          ↓
14. Perform output quality checks
```

---

# 📌 Key Design Decisions

### 1. SQLite instead of loading all reference data into pandas

The Source 2 and Source 3 datasets contain more than 10 million records combined. SQLite allows the pipeline to retrieve relevant records from disk without requiring the complete datasets to remain in memory.

### 2. Candidate generation before ML

The machine-learning model does not compare every Source 1 entity with every reference entity.

Candidate generation reduces the search space before expensive similarity calculations and model inference.

### 3. Recall-oriented candidate generation

Candidate generation prioritizes retrieving genuine matches because a missed candidate cannot be recovered by the downstream classifier.

### 4. Multiple matches are supported

The system does not enforce a one-to-one relationship. A Source 1 entity can be associated with multiple Source 2 and/or Source 3 entities.

### 5. Probability thresholding

The Random Forest model produces match probabilities. A selected threshold is used to determine which candidate relationships are retained in the final matching output.

---

# 🚀 Current Status

### Completed

* Problem understanding
* Dataset inspection
* Memory-safe processing strategy
* SQLite database construction
* SQLite indexing
* Chunk-based processing
* Candidate generation
* Candidate ranking
* Candidate recall evaluation
* Missed-match analysis
* Similarity feature engineering
* Random Forest model training
* Matching threshold selection
* Test-data inference pipeline
* Final matching-result generation
* Submission validation workflow

### Finalization

* Generate/verify `candidate_pairs.tsv`
* Run official submission validator
* Perform final output-quality checks
* Finalize repository documentation
* Push required code and outputs to GitHub

---

# 🛠️ Technologies Used

* **Python**
* **Pandas**
* **NumPy**
* **SQLite**
* **Scikit-learn**
* **Joblib**
* **Jupyter Notebook**
* **Amazon SageMaker**

---

# 📌 Notes

The raw datasets are intentionally excluded from the GitHub repository because of their size.

The repository contains the processing and matching pipeline, configuration/model artifacts where appropriate, documentation, and final submission outputs rather than the complete raw datasets.

---

## 👩‍💻 Project

**Business Entity Resolution**

Repository:

```text
NidhiPatil22/business-entity-resolution
```
