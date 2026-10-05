# 🔗 Business Entity Matching

### Cross-Source Business Record Linkage at Scale




\

A scalable pipeline for identifying records across multiple business data sources that refer to the **same real-world business**, despite differences in names, addresses, formatting, abbreviations, spelling, and country representation.

---

## 📑 Table of Contents

* [Overview](#-overview)
* [Problem Statement](#-problem-statement)
* [Dataset](#-dataset)
* [Project Pipeline](#-project-pipeline)
* [Data Understanding](#1-data-understanding)
* [Data Preprocessing](#2-data-preprocessing)
* [Candidate Generation and Blocking](#3-candidate-generation-and-blocking)
* [Similarity Feature Engineering](#4-similarity-feature-engineering)
* [Entity Matching](#5-entity-matching)
* [Why Blocking Is Important](#-why-blocking-is-important)
* [Evaluation](#-evaluation)
* [Results](#-results)
* [Technology Stack](#-technology-stack)
* [Project Structure](#-project-structure)
* [Getting Started](#-getting-started)
* [Challenges](#-challenges)
* [Progress and Roadmap](#-progress-and-roadmap)
* [AWS SageMaker](#-aws-sagemaker)
* [Author](#-author)
* [License](#-license)
* [Acknowledgements](#-acknowledgements)

---

## 📌 Overview

**Business Entity Matching**, also known as **Entity Resolution** or **Record Linkage**, is the process of determining which records from different data sources represent the same real-world entity.

In this project, the entity being matched is a **business**.

For example, the following records may refer to the same business:

| Source   | Business Name          | Address                                              |
| -------- | ---------------------- | ---------------------------------------------------- |
| Source 1 | `Orelee's Barbershop`  | `1795 Westchester Drive, High Point, NC`             |
| Source 2 | `Orelee Barbershop`    | `1795 Westchester Dr, High Point, NC`                |
| Source 3 | `Orelee's Barber Shop` | `1795 Westchester Drive, High Point, North Carolina` |

Although the records are not textually identical, their underlying information indicates that they may represent the **same real-world business**.

The project develops a multi-stage pipeline for:

```text
Data Understanding
        ↓
Data Preprocessing
        ↓
Text Normalization
        ↓
Candidate Generation / Blocking
        ↓
Similarity Feature Engineering
        ↓
Entity Matching
        ↓
Evaluation
        ↓
Final Entity Mapping
```

The datasets contain **millions of business records**, making scalability an important part of the problem.

---

## 🎯 Problem Statement

Business information collected from different sources is often inconsistent.

Common variations include:

* Different business name formats
* Spelling variations
* Abbreviations
* Different business suffixes
* Punctuation differences
* Capitalization differences
* Extra or missing spaces
* Address formatting differences
* Missing address components
* Different country representations

For example:

```text
ABC Restaurant
ABC Restaurants
A.B.C. Restaurant
ABC Rest.
```

These records may represent the same business even though exact string matching would treat them as different.

Therefore, the core question of this project is:

> **Do these two business records refer to the same real-world business?**

---

## 🧩 Dataset

The project contains three business data sources and a ground-truth dataset.

| Dataset      |   Records |
| ------------ | --------: |
| Source 1     | 2,206,821 |
| Source 2     | 5,034,616 |
| Source 3     | 5,285,603 |
| Ground Truth | 2,206,821 |

### Main Columns

| Column                        | Description                                     |
| ----------------------------- | ----------------------------------------------- |
| `business_name`               | Original business name                          |
| `business_name_normalized`    | Normalized business name used for comparison    |
| `business_address`            | Original business address                       |
| `business_address_normalized` | Normalized business address used for comparison |
| `country`                     | Original country value                          |
| `country_normalized`          | Standardized country value                      |

### Dataset Policy

> ⚠️ The raw competition datasets are **not included in this GitHub repository**. The provided datasets should be stored locally according to the competition's rules and data-usage terms.

---

# 🏗️ Project Pipeline

## 1. Data Understanding

The first stage focuses on understanding the structure, scale, and quality of the available datasets.

Activities include:

* Loading Source 1, Source 2, and Source 3
* Inspecting dataset dimensions
* Reviewing column names
* Checking data types
* Checking missing values
* Examining duplicate records
* Understanding the ground-truth mapping

The initial dataset sizes are:

```text
Source 1      : 2,206,821 records
Source 2      : 5,034,616 records
Source 3      : 5,285,603 records
Ground Truth  : 2,206,821 records
```

---

## 2. Data Preprocessing

Business records from different sources may use inconsistent formatting.

The preprocessing stage standardizes the available information before matching.

The preprocessing workflow includes:

* Handling missing values
* Standardizing text case
* Removing unnecessary punctuation
* Standardizing whitespace
* Normalizing business names
* Normalizing addresses
* Normalizing country information

### Example

```text
Original:
Orelee's Barbershop

Normalized:
orelee s barbershop
```

Normalization makes records easier to compare while preserving the information needed for entity resolution.

---

## 3. Candidate Generation and Blocking

Comparing every Source 1 record with every record in Source 2 and Source 3 would result in an extremely large number of comparisons.

Therefore, the project uses **candidate generation**, also known as **blocking**, to reduce the search space.

A blocking key used during the candidate-generation stage is:

```python
blocking_key = country_normalized + "_" + first 3 characters of business_name_normalized
```

Conceptually:

```text
Country
   +
Business Name Prefix
   ↓
Blocking Key
   ↓
Candidate Records
```

Only records that satisfy the selected blocking criteria are considered as candidate pairs for further comparison.

### Why Candidate Generation?

Without blocking:

```text
Source 1 × Source 2
+
Source 1 × Source 3
```

would produce an enormous number of possible pairs.

With blocking:

```text
Millions of Records
        ↓
Blocking
        ↓
Smaller Candidate Set
        ↓
Detailed Similarity Matching
```

This significantly reduces the amount of expensive comparison required.

---

## 4. Similarity Feature Engineering

After candidate generation, potential business pairs can be compared using multiple matching signals.

### Matching Signals

| Signal                   | Purpose                                                  |
| ------------------------ | -------------------------------------------------------- |
| Business name similarity | Captures spelling and formatting variations              |
| Address similarity       | Captures address formatting and abbreviation differences |
| Country match            | Provides an additional agreement signal                  |

Conceptually:

```text
Business Name
      │
      ├── Similarity
      │
Address
      │
      ├── Similarity
      │
Country
      │
      └── Match
           ↓
     Combined Evidence
```

Using multiple attributes is more robust than relying on a single field.

---

## 5. Entity Matching

The candidate pairs generated during blocking are evaluated using their available matching features.

Conceptually:

```text
Name Similarity
       +
Address Similarity
       +
Country Match
       ↓
Matching Evidence
       ↓
   Match Decision
       │
   ┌───┴────┐
   ↓        ↓
 Match   Non-Match
```

The final matching strategy is being developed and evaluated against the provided ground-truth data.

---

# 💡 Why Blocking Is Important

The dataset contains millions of records.

A naive pairwise approach would require comparisons approximately on the scale of:

```text
Source 1 × Source 2
+
Source 1 × Source 3
```

This creates billions of potential record pairs.

Performing detailed similarity calculations on every possible pair would be computationally expensive and difficult to scale.

Blocking addresses this problem by generating a smaller set of likely candidates before performing more expensive matching operations.

### Benefits

* ⚡ Faster processing
* 📈 Better scalability
* 💾 Lower memory requirements
* 🔍 Smaller candidate search space
* 🧮 Fewer expensive similarity calculations

A good blocking strategy should reduce the number of comparisons **without removing too many true matches**.

---

# 📊 Evaluation

The provided **Ground Truth** dataset is used to evaluate the quality of the entity-matching pipeline.

The evaluation can include:

* Precision
* Recall
* F1-score
* Accuracy
* True Positives
* False Positives
* False Negatives

### Important Entity-Resolution Metrics

In addition to conventional classification metrics, the blocking stage can be evaluated using:

**Pair Completeness**

> The proportion of true matching pairs that remain after candidate generation.

**Reduction Ratio**

> The proportion of all possible record pairs eliminated by blocking.

These metrics help determine whether the blocking strategy is both:

```text
Efficient
   +
Accurate
```

---

# 📈 Results

> 🚧 **The final matching results are currently being developed.**

Final performance metrics will be added after completing the matching and evaluation stages.

| Metric                     | Score |
| -------------------------- | ----: |
| Precision                  |   TBD |
| Recall                     |   TBD |
| F1-score                   |   TBD |
| Accuracy                   |   TBD |
| Blocking Pair Completeness |   TBD |
| Blocking Reduction Ratio   |   TBD |

### Sample Matching Example

| Record A                                      | Record B                                 | Expected Relationship | Score |
| --------------------------------------------- | ---------------------------------------- | --------------------- | ----: |
| `Orelee's Barbershop, 1795 Westchester Drive` | `Orelee Barbershop, 1795 Westchester Dr` | Match                 |   TBD |
| Additional example                            | Additional example                       | TBD                   |   TBD |

> Final examples and scores will be added after the matching model is completed.

---

# ⚙️ Technology Stack

| Category                | Technologies                                       |
| ----------------------- | -------------------------------------------------- |
| Programming Language    | Python                                             |
| Data Processing         | Pandas, NumPy                                      |
| Development Environment | JupyterLab                                         |
| Data Format             | TSV                                                |
| Text Processing         | String normalization and similarity-based matching |
| Version Control         | Git, GitHub                                        |
| Potential Extensions    | RapidFuzz, Scikit-learn, TF-IDF, Character N-grams |
| Cloud / ML Platform     | AWS SageMaker                                      |

---

# 📁 Project Structure

```text
Business-Entity-Matching/
│
├── student_resource/
│   └── dataset/
│       ├── train/
│       │   ├── train_source1.tsv
│       │   ├── train_source2.tsv
│       │   ├── train_source3.tsv
│       │   └── ground_truth.tsv
│       │
│       └── output/
│           └── preprocessed/
│
├── notebooks/
│   ├── 01_data_understanding.ipynb
│   ├── 02_preprocessing.ipynb
│   ├── 03_candidate_generation.ipynb
│   ├── 04_similarity_matching.ipynb
│   └── 05_evaluation.ipynb
│
├── src/
│   ├── preprocessing.py
│   ├── blocking.py
│   ├── matching.py
│   └── evaluation.py
│
├── requirements.txt
├── README.md
└── .gitignore
```

> **Note:** The project structure may evolve as additional matching and evaluation components are implemented.

---

# 🚀 Getting Started

## 1. Clone the Repository

```bash
git clone https://github.com/Biswabasini90/<your-repository-name>.git
cd <your-repository-name>
```

Replace `<your-repository-name>` with the actual GitHub repository name.

---

## 2. Create a Virtual Environment

### Windows

```bash
python -m venv venv
venv\Scripts\activate
```

### Linux / macOS

```bash
python3 -m venv venv
source venv/bin/activate
```

---

## 3. Install Dependencies

```bash
pip install -r requirements.txt
```

---

## 4. Add the Dataset

Place the competition-provided TSV files in:

```text
student_resource/dataset/train/
```

Expected files:

```text
train_source1.tsv
train_source2.tsv
train_source3.tsv
ground_truth.tsv
```

> ⚠️ Do not commit the raw competition datasets to GitHub unless their redistribution is explicitly permitted.

---

## 5. Launch JupyterLab

```bash
jupyter lab
```

Run the notebooks in the following logical order:

```text
01_data_understanding
        ↓
02_preprocessing
        ↓
03_candidate_generation
        ↓
04_similarity_matching
        ↓
05_evaluation
```

---

# 🔬 Challenges

Business entity resolution presents several real-world challenges.

### 1. Business Name Variations

```text
ABC Restaurant
ABC Restaurants
A.B.C. Restaurant
ABC Rest.
```

### 2. Address Variations

```text
123 Main Street
123 Main St.
123 Main St
```

### 3. Missing Information

Some records may contain incomplete names, addresses, or country information.

### 4. Duplicate Records

The same business may appear multiple times within the same source or across different sources.

### 5. Large-Scale Data

Millions of records make brute-force pairwise comparison computationally expensive.

### 6. False Matches

Two different businesses may have very similar names or addresses.

### 7. Missed Matches

Overly strict blocking or similarity thresholds can cause genuine matches to be discarded.

Therefore, the system must balance:

```text
Precision
   ↕
Recall
   ↕
Scalability
```

---

# 🗺️ Progress and Roadmap

### Current Progress

* [x] Project setup
* [x] Dataset loading
* [x] Data understanding
* [x] Initial data quality analysis
* [x] Data preprocessing
* [ ] Text normalization
* [ ] Candidate generation / blocking
* [ ] Similarity feature engineering
* [ ] Entity matching and scoring
* [ ] Evaluation against ground truth
* [ ] Final entity mapping
* [ ] Final performance analysis

### Future Improvements

Once the baseline pipeline is complete, possible improvements include:

* Advanced fuzzy matching
* Character n-gram similarity
* TF-IDF-based matching
* Phonetic matching
* Address-specific parsing
* Geographic distance features
* Machine-learning-based pair classification
* Candidate ranking
* Confidence scoring
* Human-in-the-loop verification
* Ensemble matching models
* Distributed processing using Apache Spark
* Transformer-based embeddings
* Scalable cloud deployment

---

# ☁️ AWS SageMaker

The hackathon encourages the use of **AWS SageMaker** for machine-learning workflows.

The project is currently developed and experimented with using **JupyterLab**.

AWS SageMaker can potentially support future stages such as:

* Large-scale data processing
* Model training
* Experiment tracking
* Hyperparameter tuning
* Model deployment
* Scalable inference

SageMaker can therefore be integrated as the project moves from local experimentation toward a more scalable production-oriented workflow.

---

# 👩‍💻 Author

## Biswabasini Prasad

**B.Tech — Computer Science Engineering**
**Specialization: Data Analytics & Machine Learning**

GitHub: [@Biswabasini90](https://github.com/Biswabasini90)

---

# 📜 License

This project is developed for **educational and hackathon purposes**.

The competition-provided datasets may be subject to separate terms and conditions. Please refer to the official competition rules and dataset license before redistributing any dataset or derived data.

---

# ⭐ Acknowledgements

This project was developed as part of a **Business Entity Matching / Entity Resolution Hackathon Challenge**.

The project applies practical techniques from:

* Data Engineering
* Natural Language Processing
* Record Linkage
* Entity Resolution
* Similarity Matching
* Machine Learning
* Large-Scale Data Processing

The objective is to develop a scalable and reliable system for identifying the same real-world businesses across heterogeneous data sources.
