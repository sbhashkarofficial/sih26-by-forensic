# Problem Statement

- Problem Statement Id: 26188

- Problem Statement Title: Al-Based Fake Identity & Document Screening System

## Description

Border checkpoints are bottlenecks. Security personnel currently rely on visual inspection and basic database queries to verify passports, visas, and IDs. This manual process takes minutes per passenger and completely fails to detect sophisticated digital or physical tampering

## What we need to build?

A web application with 

- **frontend dashboard** : for officials who verifies documents

- **backend server** : for document processing and calculating the risk score for each document processed.

- **database** : for storing the data of each processed document. *To create a digital trail for investigations and intelligence analysis.*

> A digital trail (or audit trail) means the system must maintain a permanent, searchable database record of every single document processed and every decision made by the border agents.

The application is divided in 4 parts:

### Module 1: OCR Extraction

Objectinve : Converts the image of the document into structured data(json).

For instance, ocr extraction on a passport would produce a json data containing the fields: 

- Name

- Passport Number

- Nationality

- Date of birth, and so on.

### Module 2: Document Validation:

Objective : Verify whether the extracted information follows official document standards. 

For example: checking the expiration date against the current date

=========

### Module 3: Tampering Detection (Core AI Innovation) --- most important part

Objective: Detects if the document was physically altered or photoshopped.

What to check for:

- Photo replacement

- text manipulation

- forgery stamp

- image metadata analysis

### Module 4: Face Verification

Objective: Ensure document owner matches the presented individual.

### Overall Goals

- Reduce document verification time from several minutes to a few seconds.

- Imporve detection of forged and tempered documents.

- Standardize screening decisions.

- Create a digital trail for investigations and intelligence analysis.
