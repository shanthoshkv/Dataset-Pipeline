# Dataset Pipeline

### Local-First Document-to-Dataset Pipeline for LLM Fine-Tuning

**Dataset Pipeline** is a production-oriented, fully local pipeline that converts technical documents and structured data into high-quality datasets for LLM fine-tuning, reasoning training, and preference optimization.

The idea behind the project is fairly simple: getting documents into an LLM training dataset is easy. Getting them into a dataset that is actually useful is much harder.

Raw documents contain tables, formulas, repeated information, poorly formatted text, scanned pages, irrelevant sections, malformed model outputs, and near-duplicate questions. A simple PDF-to-text script does not solve these problems.

This project approaches dataset generation as an engineering pipeline.

It ingests documents, preserves their structure, creates semantic chunks, generates training examples locally using Ollama, validates every example, removes duplicates, enriches the dataset with reasoning and preference data, and finally produces training-ready exports along with detailed quality reports.

**No mandatory cloud APIs. No external inference. No API keys required.**

Everything can run locally.

## Live Demo

**Project Website:** https://aboutkvs.vercel.app/dataset_pipeline.html

---

# Why I Built This

Most synthetic dataset pipelines follow a pattern similar to:

```text
Document
   ↓
Extract Text
   ↓
Ask LLM to Generate Questions
   ↓
Save JSON
```

That works for a demo.

It becomes problematic when the dataset is large, the source material is technical, or the resulting data is going to be used for actual fine-tuning.

A generated answer can be hallucinated.

A question can be meaningless.

Two questions can be almost identical.

A model can truncate its response.

A PDF can contain formulas or tables that get destroyed during extraction.

A scanned document may contain almost no machine-readable text.

A generated dataset can look large while containing very little useful training signal.

Dataset Pipeline was built to address these problems systematically.

The objective is to make the dataset itself the product of a controlled engineering process rather than simply the output of an LLM prompt.

---

# Pipeline Overview

The system uses a seven-stage workflow:

```text
                    ┌──────────────────────┐
                    │   Source Documents   │
                    │ PDF DOCX XLSX PPTX   │
                    │ HTML TXT CSV         │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ 1. Ingestion         │
                    │ Structure-aware      │
                    │ extraction           │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ 2. Semantic Chunking │
                    │ 300–400 tokens       │
                    │ 50 token overlap     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ 3. QA Generation     │
                    │ Local Ollama models  │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ 4. Quality Validation│
                    │ Six scoring criteria │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ 5. Deduplication     │
                    │ Embedding similarity │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ 6. Enrichment        │
                    │ CoT / DPO / Multi-hop│
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ 7. Export + Reports  │
                    │ JSON / JSONL / HTML  │
                    └──────────────────────┘
```

Each stage has a defined responsibility and can be tested, replaced, or optimized independently.

---

# Supported Input Formats

The pipeline supports seven major input categories.

| Format | Extraction                  |
| ------ | --------------------------- |
| PDF    | pdfplumber + PyMuPDF        |
| DOCX   | python-docx                 |
| XLSX   | openpyxl                    |
| PPTX   | python-pptx                 |
| HTML   | HTML text extraction        |
| TXT    | Direct structured ingestion |
| CSV    | Header-aware row extraction |

The important part is that these are not treated as identical text files.

Each format has its own extraction path so that as much useful structure as possible is retained before chunking.

---

# 1. Document Ingestion

The first stage converts heterogeneous source material into normalized internal records.

For PDFs, the pipeline uses:

* `pdfplumber`
* PyMuPDF fallback
* Scanned-page detection
* VLM-based visual extraction

For Office files:

* `python-docx`
* `python-pptx`
* `openpyxl`

Tables, paragraphs, slide notes, sheet structure, and other useful document information are retained where possible.

This is particularly important for technical documents where destroying a table or equation during extraction can completely change the meaning of the resulting dataset.

---

# 2. Semantic Chunking

Instead of simply splitting documents every N characters, the pipeline creates semantic chunks.

The default configuration is:

```text
Chunk size:       300–400 tokens
Overlap:           50 tokens
```

The chunking system attempts to preserve:

* Sentence boundaries
* Equations
* LaTeX blocks
* Tables
* Logical sections
* Technical context

This prevents situations where an equation is split in half or a table row is separated from the information needed to understand it.

The goal is not simply to produce chunks of equal size.

The goal is to produce chunks that still make sense when presented independently to a language model.

---

# 3. Local QA Generation

Question-answer generation is performed locally through **Ollama**.

This removes the requirement for cloud inference APIs and allows the entire dataset-generation process to operate on the user's own hardware.

The pipeline can detect locally available Ollama models and route models according to their intended role.

Possible roles include:

* QA generation
* Categorization
* Visual extraction
* Dataset enrichment

This makes the system flexible across different local model configurations.

For example, a lightweight model can handle classification while a larger model handles complex QA generation.

---

# Robust JSON Recovery

One of the less obvious problems with automated dataset generation is that LLMs do not always return valid JSON.

A model may produce:

```text
Here is the requested JSON:

{
    "question": "...",
    "answer": "..."
}

Hope this helps!
```

Or it may return malformed JSON, partially truncated structures, or additional text around the expected schema.

Instead of allowing one malformed response to break a long-running generation job, the pipeline uses a four-stage recovery process:

```text
Strict JSON Parse
       ↓
Lenient Parse
       ↓
Regex Extraction
       ↓
Schema Reconstruction
```

This makes the generation process substantially more resilient during large batch runs.

---

# 4. Quality Control

This is one of the most important parts of the project.

Every generated question-answer pair is evaluated across six dimensions.

## Groundedness

Does the answer actually come from the source context?

This helps prevent the model from introducing information that was not present in the document.

**Weight: 25 points**

---

## Length Control

Extremely short answers often contain insufficient information, while excessively long answers can contain unnecessary or low-density content.

**Weight: 20 points**

---

## Hallucination Markers

The system looks for signals associated with uncertain, refusal-style, or unsupported outputs.

**Weight: 20 points**

---

## Truncation Detection

The pipeline detects incomplete generations such as:

```text
The main advantage of this method is...
```

where the response clearly ends prematurely.

**Weight: 15 points**

---

## Question Structure

The generated prompt must actually be a usable question rather than a malformed statement or degenerate output.

**Weight: 10 points**

---

## Answer Fit

The answer should respond to the question rather than simply copying or repeating the prompt.

**Weight: 10 points**

---

# Quality Score

The six dimensions produce a total score out of 100.

```text
Score ≥ 60
    ↓
Pass
    ↓
Deduplication

Score 40–59
    ↓
Discarded as low quality

Score < 40
    ↓
Hard failure
```

This creates a clear quality gate before examples are allowed into the final dataset.

The thresholds and diagnostics are also recorded so that the process can be audited after a run.

---

# 5. Semantic Deduplication

A large synthetic dataset can easily contain thousands of questions that are technically different but semantically almost identical.

For example:

```text
What is the purpose of regenerative cooling?

Why is regenerative cooling used?

What is regenerative cooling used for?

Why do rocket engines use regenerative cooling?
```

Keeping all four can artificially inflate the dataset without adding much new information.

The pipeline uses embedding-based similarity to identify near-duplicate records.

The default similarity threshold is:

```text
0.85 cosine similarity
```

The primary embedding model is:

```text
BAAI/bge-m3
```

with:

```text
all-MiniLM-L6-v2
```

available as a lightweight fallback.

Exact matching is also used as a final safety mechanism.

---

# 6. Dataset Enrichment

The pipeline goes beyond conventional question-answer generation.

Three enrichment modes are supported.

## Chain-of-Thought Data

The pipeline can generate reasoning traces alongside final answers.

A record can therefore contain separate reasoning and answer fields rather than only:

```json
{
  "question": "...",
  "answer": "..."
}
```

This makes the generated data suitable for experimentation with reasoning-oriented fine-tuning workflows.

Output:

```text
dataset_hf_cot.jsonl
```

---

# DPO Preference Data

The pipeline can also generate preference pairs.

Instead of only producing a correct answer, it creates:

```text
Chosen Answer
Rejected Answer
```

Controlled rejection modes can include:

* Hallucination
* Incomplete answer
* Incorrect formula
* Unsupported reasoning

This creates preference data suitable for DPO-style alignment experiments.

Output:

```text
dataset_hf_dpo.jsonl
```

---

# Multi-Hop Reasoning

Some questions cannot be answered from a single chunk.

The pipeline uses embedding similarity to identify related chunks that are not necessarily adjacent in the original document.

These chunks can then be combined to create questions requiring synthesis across multiple source passages.

Examples include:

* Causal questions
* Comparative questions
* Inferential questions
* Multi-step reasoning

This is particularly useful for technical documents where the information needed to answer a question may be distributed across different sections.

---

# Visual Document Extraction

Technical documents frequently contain useful information inside:

* Figures
* Scanned pages
* Diagrams
* Screenshots
* Slide images

A text-only extraction pipeline would simply lose this information.

Dataset Pipeline therefore includes a visual extraction path using a vision-capable local model.

The VLM can process visual content and extract information that can subsequently enter the normal dataset-generation pipeline.

This allows scanned and image-heavy documents to contribute training examples rather than being treated as empty pages.

---

# 7. Dataset Export

A single pipeline run produces multiple training-ready formats.

## ShareGPT

```text
dataset_hf_sharegpt.jsonl
```

Conversation-style records with role-based messages.

Useful for:

* Unsloth
* TRL
* Conversational fine-tuning

---

## Alpaca

```text
dataset_hf_alpaca.jsonl
```

Instruction, input, and output records suitable for a wide range of fine-tuning workflows.

---

## Chain-of-Thought

```text
dataset_hf_cot.jsonl
```

Separate reasoning and answer fields for reasoning-oriented experiments.

---

## DPO

```text
dataset_hf_dpo.jsonl
```

Chosen and rejected responses for preference optimization.

---

## Commercial JSON

```text
dataset_commercial.json
```

A metadata-rich representation containing information such as:

* Domain
* Difficulty
* Timestamp
* Source filename
* Provenance
* Dataset metadata

---

## Commercial JSONL

```text
dataset_commercial.jsonl
```

A streaming-friendly version of the commercial schema designed for larger datasets and pipeline-based processing.

---

# Quality Reports

Dataset generation should not end when the JSON file is created.

Every run also generates two quality reports.

## HTML Report

```text
quality_report.html
```

Provides visual analysis including:

* Score distributions
* Domain distribution
* Difficulty breakdown
* Dataset health
* Quality statistics
* Dataset composition

---

## JSON Diagnostics

```text
quality_report.json
```

Provides machine-readable diagnostics suitable for:

* CI pipelines
* Automated quality gates
* Auditing
* Experiment tracking
* Downstream analytics

This makes the pipeline much easier to integrate into a larger ML workflow.

---

# Architecture

The major backend components are intentionally separated by responsibility.

```text
dataset_pipeline/
│
├── parser.py
│   └── Robust JSON recovery
│
├── chunker.py
│   └── SemanticChunker
│
├── tokens.py
│   └── TokenCounter
│
├── registry.py
│   └── Ollama ModelRegistry
│
├── vision.py
│   └── VLM visual extraction
│
└── stats.py
    └── DatasetStatistics
```

This modular architecture means that individual components can be improved without rewriting the entire pipeline.

For example, the embedding model can be replaced without changing document ingestion, while the QA model can be changed without modifying the validation system.

---

# Technology Stack

## Local AI

### Ollama

Used for local LLM inference and model orchestration.

### sentence-transformers

Used for semantic embeddings, similarity calculations, and chunk relationships.

### BAAI/bge-m3

Primary embedding model for semantic similarity and deduplication.

### all-MiniLM-L6-v2

Lightweight embedding fallback.

---

## Document Processing

### pdfplumber

Primary PDF text extraction.

### PyMuPDF

Fallback and robust PDF processing.

### python-docx

DOCX parsing.

### python-pptx

PowerPoint extraction.

### openpyxl

Excel processing.

### pdf2image + Pillow

Scanned-page conversion and visual preprocessing.

---

## Validation and Analytics

### tiktoken

Token counting and safe output shaping.

### NumPy

Numerical processing.

### scikit-learn

Cosine similarity and statistical processing.

### tenacity

Retry handling for unreliable local model responses.

### matplotlib

Quality-report visualization.

---

# Local-First Architecture

Privacy is a major design requirement of the project.

The pipeline is designed so that source documents do not need to leave the local machine.

```text
                LOCAL MACHINE
┌──────────────────────────────────────────┐
│                                          │
│ Documents                                │
│    ↓                                     │
│ Extraction                               │
│    ↓                                     │
│ Chunking                                 │
│    ↓                                     │
│ Local Ollama                             │
│    ↓                                     │
│ Local Embeddings                         │
│    ↓                                     │
│ Validation                               │
│    ↓                                     │
│ Deduplication                            │
│    ↓                                     │
│ Dataset Export                           │
│                                          │
└──────────────────────────────────────────┘
```

There are:

* No mandatory cloud inference APIs
* No external embedding APIs
* No mandatory API keys
* No required cloud storage
* No requirement to upload sensitive documents

This makes the architecture particularly interesting for:

* Research laboratories
* Engineering organizations
* Internal knowledge bases
* Proprietary technical documentation
* Sensitive datasets
* Air-gapped environments

---

# Engineering Design Principles

The project follows several principles throughout the implementation.

### 1. Local First

Models and embeddings should run locally whenever possible.

### 2. Fail Gracefully

A malformed model response should not terminate a large dataset-generation run.

### 3. Preserve Structure

Tables, equations, paragraphs, and visual information should survive the extraction process.

### 4. Quality Before Quantity

A smaller dataset with reliable examples is more valuable than a huge dataset filled with duplicates and hallucinations.

### 5. Observable Processing

Every run should produce enough diagnostics to understand what happened.

### 6. Modular Components

Each subsystem should have a clear responsibility and be independently replaceable.

---

# Example End-to-End Workflow

A typical run looks like this:

```text
Research Paper / Technical Manual
              ↓
        Format Detection
              ↓
     Structure-Aware Extraction
              ↓
       Semantic Chunking
              ↓
        Local QA Generation
              ↓
       JSON Error Recovery
              ↓
        Quality Evaluation
              ↓
       Low-Quality Filtering
              ↓
      Embedding Deduplication
              ↓
       Dataset Enrichment
         ↙       ↓       ↘
       CoT      DPO     Multi-Hop
         ↘       ↓       ↙
          Final Dataset
              ↓
     JSON / JSONL Exports
              ↓
      HTML + JSON Reports
```

The result is not just a collection of generated questions.

It is a dataset accompanied by information about how that dataset was produced and how its quality was measured.

---

# Useful Fine-Tuning Workflows

The generated outputs are designed to work with common open-source training ecosystems.

Potential downstream workflows include:

```text
Dataset Pipeline
      ↓
ShareGPT / Alpaca / CoT / DPO
      ↓
Hugging Face Dataset
      ↓
Unsloth / TRL
      ↓
Fine-Tuning
      ↓
Evaluation
      ↓
Deployment
```

This makes the pipeline suitable as the data-engineering layer before model training.

---

# Example Applications

The system can be used to generate datasets from many different types of technical material.

### Aerospace

* Research papers
* Propulsion manuals
* CFD documentation
* Flight-test reports
* Turbomachinery literature
* Structural analysis documents

### Engineering

* Design manuals
* Technical specifications
* Standards
* Laboratory reports
* Equipment documentation

### Research

* Academic papers
* Lecture notes
* Experimental reports
* Scientific datasets

### Internal Knowledge

* Company documentation
* Internal manuals
* Engineering procedures
* Product documentation

The local architecture is particularly useful when these documents cannot be uploaded to third-party AI services.

---

# Key Parameters

The current default operating parameters include:

| Parameter               |        Default |
| ----------------------- | -------------: |
| Semantic chunk size     | 300–400 tokens |
| Chunk overlap           |      50 tokens |
| Deduplication threshold |           0.85 |
| Minimum quality score   |       60 / 100 |
| Quality dimensions      |              6 |
| Pipeline stages         |              7 |
| Input formats           |             7+ |
| Export/report files     |              8 |

These values are configurable and can be adjusted depending on document type, model capability, and desired dataset characteristics.

---

# What I Learned Building This

The biggest lesson from this project was that **dataset generation is much more difficult than dataset formatting**.

Generating 10,000 question-answer pairs is relatively easy.

Generating 10,000 useful question-answer pairs is a completely different problem.

The difficult parts are often not the LLM calls themselves. They are the engineering around them:

* Extracting information correctly
* Preserving technical structure
* Controlling token lengths
* Recovering malformed model responses
* Detecting hallucinations
* Removing duplicates
* Measuring dataset quality
* Maintaining provenance
* Handling large batch jobs
* Making the entire process reproducible

That is what this project is primarily designed around.

---

# Future Improvements

Some directions I would like to explore further include:

* Automatic dataset balancing across domains
* More sophisticated factual verification
* Model-based judge ensembles
* Automated difficulty calibration
* Dataset versioning
* Experiment tracking
* Incremental document ingestion
* Better table and equation understanding
* More advanced multi-hop generation
* Automated contamination detection
* Fine-tuning evaluation directly from generated reports
* Distributed local inference across multiple machines

---

# Project Status

The current implementation includes the complete seven-stage workflow:

* Multi-format ingestion
* Semantic chunking
* Local Ollama generation
* Robust JSON recovery
* Six-dimensional quality scoring
* Embedding-based deduplication
* CoT enrichment
* DPO preference generation
* Multi-hop question generation
* VLM visual extraction
* Multiple dataset exports
* HTML quality reporting
* JSON diagnostics

The project is designed as a foundation that can be extended into a larger local dataset-generation and fine-tuning platform.

---

# Live Project

Explore the complete architecture and interactive project overview:

**https://aboutkvs.vercel.app/dataset_pipeline.html**

---

## Author

**Shanthosh K V**

Aerospace Engineering Student
RV College of Engineering, Bengaluru

Interested in aerospace engineering, propulsion, turbomachinery, CFD, computational engineering, machine learning, and building practical engineering software.

---

## License

© 2026 Shanthosh K V. All rights reserved.
