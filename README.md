# Business Entity Resolution

A memory-efficient pipeline for identifying matching business entities across multiple large-scale business datasets.

## 📌 Overview

This project focuses on **Business Entity Resolution (BER)** across three different data sources.

Given a business record from **Source 1**, the goal is to identify all corresponding business entities in **Source 2** and **Source 3**, even when the same business is represented differently across datasets.

For example:

**Source 1**

Shree Ganesh Restaurant Pvt Ltd  
12 MG Road, Pune

↓

**Source 2**

Shree Ganesh Restaurant  
12 Mahatma Gandhi Road, Pune

**Source 3**

Shree Ganesh Restro Pvt. Ltd.  
12 MG Rd, Pune

Although the records are written differently, they may represent the same real-world business.

---

## 🎯 Objective

For every Source 1 business, identify all matching entities from Source 2 and Source 3.

The final system is expected to produce relationships of the form:

**Source 1 Entity → Matching Source 2 / Source 3 Entities**

An important characteristic of this problem is that one Source 1 business can have **multiple valid matches**. Therefore, the pipeline cannot assume a one-to-one relationship.

---

## 🧩 Challenges

The datasets contain several variations that make direct matching difficult:

- Different business-name formats
- Address variations
- Abbreviations
- Spelling variations
- Different representations across data sources
- Multiple matches for a single Source 1 entity
- Very large datasets

The combined size of Source 2 and Source 3 makes exhaustive pairwise comparison computationally expensive and memory-intensive.

---

## 📊 Dataset Scale

The current processed datasets contain:

| Source | Records |
|--------|---------:|
| Source 2 | 5,034,616 |
| Source 3 | 5,285,603 |

This results in more than **10 million records** across the two large source datasets.

Loading the complete datasets into pandas simultaneously is not practical in a memory-constrained environment.

---

## 🧠 Memory-Safe Architecture

The initial approach of loading the large datasets directly into pandas caused the SageMaker kernel to run out of memory.

To solve this, the pipeline was redesigned around **SQLite-based disk storage and indexing**.

### Current Pipeline

Source 2 + Source 3  
↓  
SQLite Database  
↓  
Indexed Search  
↓  
Source 1  
↓  
Candidate Generation  
↓  
Candidate Ranking  
↓  
Similarity Features  
↓  
ML Matching  
↓  
Final Entity Matches

The large datasets are processed in **25,000-row chunks**, allowing the system to operate without loading the complete datasets into RAM.

---

## 🗄️ SQLite Database

The large Source 2 and Source 3 datasets have been imported into SQLite.

### Completed

- Source 2 imported
- Source 3 imported
- SQLite database created
- Required indexes created
- Chunk-based processing implemented
- Large-scale data remains on disk instead of being loaded entirely into memory

The SQLite database should not be unnecessarily rebuilt once it has been created.

---

## 🔎 Candidate Generation

Comparing every Source 1 business against more than 10 million records would be computationally expensive.

Therefore, the pipeline first performs **candidate generation**.

Instead of asking:

> Which of the 10+ million records is the match?

the system first asks:

> Which records are plausible candidates?

Current candidate-generation signals include:

- Country matching
- Business-name prefix matching
- First-token matching
- Address-token overlap
- Multiple blocking conditions

This significantly reduces the number of records that need to be examined in later stages.

---

## 📈 Candidate Recall Evaluation

Candidate generation was evaluated using a subset of Source 1 records.

The evaluation identified:

- **365 actual matching relationships**
- **320 matches successfully retrieved**
- **45 matches initially missed**

This resulted in a baseline candidate recall of approximately:

**87.67%**

### Why Recall Matters

Candidate generation is designed to prioritize **recall**.

If a genuine match is not included in the candidate set, a later ML model cannot recover it.

Therefore, improving candidate recall is an important step before training the final matching model.

---

## 🔍 Missed-Match Analysis

The initially missed matches were analyzed to understand why the existing blocking conditions failed to retrieve them.

The analysis showed two major categories:

| Reason | Count |
|--------|------:|
| Block found but candidate ranked out | 37 |
| No existing block hit | 8 |
| **Total** | **45** |

This analysis helps identify weaknesses in the current candidate-generation and ranking strategy.

The next improvements focus on increasing the probability that genuine matches enter the candidate set while keeping the search computationally manageable.

---

## 🏗️ Current Development Stage

The project has currently completed the major data-processing and candidate-generation infrastructure.

### Completed

- Problem understanding
- Dataset inspection
- Memory-safe processing strategy
- SQLite database construction
- SQLite indexing
- Chunk-based processing
- Candidate generation
- Candidate ranking
- Candidate recall evaluation
- Missed-match analysis
- Initial candidate-generation improvements

### In Progress

- Candidate recall validation
- Similarity feature engineering

### Upcoming

- Training-data construction
- ML model training
- Model evaluation
- Threshold tuning
- Test-data inference
- Final matching generation
- Submission generation
- Official validation
- Final submission packaging

---

## 🤖 Machine Learning Stage

The project has **not yet started final ML model training**.

The current work primarily focuses on preparing a reliable candidate set and building the preprocessing pipeline.

Once candidate generation provides sufficiently high recall, candidate pairs can be used to construct training data.

The training data will be derived from the available dataset and candidate relationships, with examples representing:

**Candidate Pair → Match / Non-Match**

Similarity features can then be calculated for each pair, such as:

- Business-name similarity
- Address similarity
- Country match
- Token overlap
- Length-based features
- Other entity-level similarity signals

These features will be provided to the ML model during the subsequent matching stage.

---

## 🔮 Planned End-to-End Pipeline

The intended final workflow is:

**Source 1 Business**

↓

**Candidate Generation**

↓

**Candidate Ranking**

↓

**Similarity Feature Extraction**

↓

**ML Model**

↓

**Match / Non-Match Prediction**

↓

**Final Entity Matches**

↓

**Submission Files**

↓

**Official Validation**

---

## 📁 Project Structure

```text
business-entity-resolution/
│
├── code/
│   └── business_entity_resolution/
│       ├── src/
│       │   ├── memory_safe_pipeline.ipynb
│       │   └── preprocessing.py
│       │
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
