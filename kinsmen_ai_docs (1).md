# Kinsmen AI Documentation
## Intelligent Data Extraction & Ingestion from Engineering Drawings

---

### Table of Contents
1. [Overview](#overview)
2. [Data Ingestion Workflow](#data-ingestion-workflow)
3. [Pipeline Execution Logic](#pipeline-execution-logic)
4. [Stage Processes](#stage-processes)
5. [Stage Execution Mechanism](#stage-execution-mechanism)
6. [Post-Processing & Upload](#post-processing--upload)
7. [Next Additions](#next-additions)

---

## Overview

Kinsmen AI is a pipeline-based system designed to extract structured data from engineering drawings and ingest it into downstream systems. It handles drawing preprocessing, intelligent data extraction, and structured upload in two primary phases:

- **IDEA** -- Intelligent Data Extraction & Analysis
- **Upload Stages** -- Structured data ingestion into registers

---

## Data Ingestion Workflow

### Step-by-Step Flow

#### 1. File Upload
- Engineering drawings (DWG, PDF) are uploaded via the Kinsmen AI UI
- Files are stored in Azure Blob Storage
- The files are also synced to an Azure File Share, mounted to an Azure VM for processing

#### 2. Preprocessing (on Azure VM)
- **DWG to PDF Conversion**: All `.dwg` files are converted to `.pdf` and saved under the `/pdfs` folder
- **PDF to PNG Conversion**: PDFs are converted to `.png` files and stored in the `/pngs` folder
- **DWG to PNG Conversion**: Optional direct conversion of `.dwg` files to `.png`

#### 3. Minibatch Creation
- PNG files are grouped into minibatches (typically size 100)
- Each minibatch goes through a scheduled pipeline of stages

---

## Pipeline Execution Logic

Each minibatch (MB) is processed in a time-step-based schedule. A timestep represents the parallel execution unit of stage processes.

### Scheduling Pattern:
```
Timestep 1: MB1 - SP1
Timestep 2: MB2 - SP1, MB1 - SP2
Timestep 3: MB3 - SP1, MB2 - SP2, MB1 - SP3
...
```

Execution continues until all minibatches complete all stage processes. Parallelism is applied at each timestep for efficient processing.

---

## Stage Processes

### IDEA Stage Processes

#### 1. docClassification
**Purpose**: Classify the PNG images into different document types

**Process**: 
Classify the PNG images into drawing, drawing with no title block, document, or blank categories.

**Input**: PNG images

**Output**: Document class classification

**Categories**:
- Drawing (with title block)
- Drawing with no title block  
- Document
- Blank

---

#### 2. hashGeneration
**Purpose**: Generate unique image hashes for duplicate detection and content comparison

**Process**: 
Generate image hash from images for content identification and duplicate detection.

**Input**: PNG images

**Output**: Image hash values

---

#### 3. ocrExtract
**Purpose**: Extract text and structured data from images using Azure Document Intelligence

**Process**: 
Generate JSON from the images using Azure Document Intelligence service for text recognition and layout analysis.

**Input**: PNG images

**Output**: Extracted JSON containing text and layout information

---

#### 4. semSegmentation
**Purpose**: Semantic segmentation identifies different regions within engineering drawings

**Process**: 
Semantic segmentation identifies different regions including title blocks, drawings, text areas, and other document sections.

**Input**: PNG images

**Output**: Segmented PNG images with identified regions

**Segmentation Areas**:
- Title blocks
- Drawing areas  
- Text areas
- Other document regions

---

#### 5. labelExtractionV2
**Purpose**: Intelligent title block extraction and metadata processing using AI

**Process**: 
- Find title block region using segmentation data
- Refine bounding box coordinates for precision
- Extract ROI (Region of Interest) from image
- Send ROI image, filtered OCR text, and block data to LLM
- Use dynamic prompts for flexible metadata extraction
- Cache results to avoid reprocessing

**Input**: PNG images, segmentation data, OCR results

**Output**: 
- **Metadata JSON**: Extracted title block information
- **File Tracking**: Input/output file paths for pipeline integration  
- **Usage Statistics**: LLM processing costs and model information

---

#### 6. symbolDetectionPnId
**Purpose**: Detect and identify symbols in engineering drawings using YOLO V10

**Process**: 
Detects the symbols in the image using YOLO V10 model. This process is only applied to drawings.

**Input**: PNG images (drawings only)

**Output**: Extracted Symbols JSON with coordinates and classifications

---

#### 7. postProcessingPdfCreation
**Purpose**: Create searchable PDF documents from original PDFs

**Process**: 
Create searchable PDFs by adding OCR text layer to original PDF documents.

**Input**: PDF files

**Output**: Searchable PDF files with OCR layer

---

#### 8. postProcessingTagDataExtraction
**Purpose**: Extract valid engineering tags from files for asset identification

**Process**: 
Extract the valid tags from the files including DWG files, PDFs, and symbol tags.

**Input**: 
- DWG files
- PDF files  
- Symbol tags from symbol detection

**Output**: Extracted tag data

---

#### 9. postProcessingExportLabels
**Purpose**: Consolidate and export label extraction results

**Process**: 
This stage consolidates and exports label extraction results from multiple pages into a single, structured output file for further processing.

**Input**: 
- Label extraction output from labelExtractionV2
- DWG files
- PDF files

**Output**: Consolidated extracted JSON

---

#### 10. dwgBlockExtractor
**Purpose**: Extract and analyze DWG block references from AutoCAD files

**Process**: 
This stage extracts and analyzes DWG block references from AutoCAD files to identify blocks that contain titleblock information and tag values for engineering documents.

**Input**: DWG files

**Output**: 
- **JSON File**: Complete block data with all attributes
- **Filtered Text File**: Only blocks containing titleblock information

---

#### 11. detectDuplicateContentV2
**Purpose**: Identify and categorize duplicate content in engineering documents

**Process**: 
This stage identifies and categorizes duplicate content in engineering documents using image hashing and OCR text comparison to detect exact and near-duplicate files.

**Input**: 
- **PNG Images**: Converted from PDFs/DWGs
- **Image Hashes**: Pre-computed hash values from hashGeneration stage
- **OCR Data**: Text extraction results for content comparison
- **LSH Index**: Locality-Sensitive Hashing index for efficient similarity search

**Output**: 
- **Exact Duplicates**: Files with identical content
- **Near Duplicates**: Files with similar content but differences  
- **Graph Relationships**: File relationships stored in Neo4j database

---

#### 12. dataCleanup
**Purpose**: Perform intelligent data cleanup, grouping, and merging

**Process**: 
This stage performs intelligent data cleanup, grouping, and merging of titleblock information from engineering documents using AI-powered analysis and validation.

**Input**: 
- **Label Files**: Extracted titleblock data from postProcessingExportLabels stage
- **Tag Files**: Tag extraction results from postProcessingTagDataExtraction stage
- **Block Files**: DWG block information from dwgBlockExtractor stage

**Output**: 
- **Labels Output**: Consolidated titleblock information in JSON format
- **Tags Output**: Preserved tag data with source file mapping

---

#### 13. ddtClassification
**Purpose**: Intelligent classification of engineering documents

**Process**: 
This stage performs intelligent classification of engineering documents by determining their Discipline and Document Type using AI analysis of titleblock data and DWG block information.

**Input Dependencies**: 
- **Label Files**: Cleaned titleblock data from dataCleanup stage
- **Block Files**: DWG block information from dwgBlockExtractor stage  
- **Reference Data**: Discipline and document type master data from tenant configuration

**Output**: 
- **Classification Data**: Discipline and document type for each record
- **DDT IDs**: Unique identifiers for discipline-document type combinations
- **Enriched Labels**: Original data combined with classification results

---

### Upload Stage Processes

- **composeSchema**
- **splitMergeGroupFiles** 
- **uploadItemRegister**
- **uploadContentRegister**
- **uploadTagRegister**
- **associateDupItemRevision**

---

## Stage Execution Mechanism

Each Stage Process is implemented with two distinct triggers:

### 1. HTTP Trigger (create_job)
- Initiates the execution of a stage
- Retrieves all required files for the minibatch from Redis Cache or Graph DB
- Creates a task list --- one task per file
- Stores the task list and enqueues messages for each task to the Azure Service Bus Queue

### 2. Azure Service Bus Trigger (queue_job)
- Triggered when messages arrive in the scheduled service bus queue
- Executes the actual business logic for each stage process
- Processes the file and updates the graph/cache with results

---

## Post-Processing & Upload

After all IDEA stages are completed, users can initiate upload stages via the UI. These stages register the extracted content into appropriate backend systems like Item Register and Tag Register.

---

