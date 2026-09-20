# 📧 AWS-Based Outlook Email Document Processing & Classification System

## 📌 Project Overview

This project is designed to process **500+ emails received through Microsoft Outlook**, automatically extract their attachments, store the documents in Amazon S3, extract text using Amazon Textract, classify the documents using Amazon Bedrock, and separate documents based on a **90% confidence threshold**.

The system uses **batch processing** so that hundreds of emails do not have to be processed strictly one by one.

### Main Objective

The system should:

1. Read emails from Microsoft Outlook.
2. Extract email attachments.
3. Process incoming emails efficiently in batches.
4. Store attachments in Amazon S3.
5. Extract text from documents using Amazon Textract.
6. Classify documents using Amazon Bedrock.
7. Check the classification confidence.
8. Automatically route high-confidence documents.
9. Send low-confidence documents for manual review.
10. Store processing metadata and results in DynamoDB.

---

# 🏗️ System Architecture

```text
┌──────────────────────────────────────────────────────────────┐
│                        MICROSOFT 365                         │
│                          OUTLOOK                             │
│                         500+ Emails                          │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
                 ┌────────────────────────┐
                 │  Microsoft Graph API    │
                 │                        │
                 │ Read emails             │
                 │ Read attachments        │
                 │ Get email metadata      │
                 └────────────┬───────────┘
                              │
                              ▼
                 ┌────────────────────────┐
                 │      Amazon SQS         │
                 │                        │
                 │ Queue incoming emails   │
                 │ Buffer workload         │
                 └────────────┬───────────┘
                              │
                         BATCHING
                    e.g. up to 10 messages
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
           ┌────────┐    ┌────────┐    ┌────────┐
           │Lambda  │    │Lambda  │    │Lambda  │
           │Execution│   │Execution│   │Execution│
           │   #1   │    │   #2   │    │   #3   │
           │Batch 1 │    │Batch 2 │    │Batch 3 │
           └────┬───┘    └────┬───┘    └────┬───┘
                └─────────────┼─────────────┘
                              ▼
                 ┌────────────────────────┐
                 │       Amazon S3         │
                 │                        │
                 │ Store attachments       │
                 │ PDF / DOCX / JPG / PNG  │
                 └────────────┬───────────┘
                              │
                              ▼
                 ┌────────────────────────┐
                 │    Amazon Textract      │
                 │                        │
                 │ OCR                     │
                 │ Text extraction         │
                 │ Forms / Tables          │
                 └────────────┬───────────┘
                              │
                              ▼
                 ┌────────────────────────┐
                 │     Amazon Bedrock      │
                 │                        │
                 │      Nova Micro         │
                 │ Document Classification│
                 └────────────┬───────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │ Confidence Check │
                    └────────┬─────────┘
                             │
                   ┌─────────┴─────────┐
                   │                   │
                >= 90%               < 90%
                   │                   │
                   ▼                   ▼
          ┌─────────────────┐  ┌─────────────────┐
          │ HIGH CONFIDENCE │  │  LOW CONFIDENCE │
          │                 │  │                 │
          │ S3/invoice      │  │ S3/manual-review│
          │ S3/resume       │  │                 │
          │ S3/contract     │  │ Human Review    │
          │ S3/receipt      │  │                 │
          └────────┬────────┘  └────────┬────────┘
                   └──────────┬─────────┘
                              ▼
                    ┌──────────────────┐
                    │    DynamoDB      │
                    │                  │
                    │ Category         │
                    │ Confidence       │
                    │ S3 Path          │
                    │ Status           │
                    └──────────────────┘
```

---

# 🔄 Complete Workflow

## Step 1 — Emails arrive in Outlook

The company receives more than 500 emails through Microsoft Outlook.

For example:

```text
Email 1 → Invoice.pdf
Email 2 → Resume.pdf
Email 3 → Contract.pdf
Email 4 → PurchaseOrder.pdf
...
Email 500 → Receipt.pdf
```

The first problem is:

> How can we automatically read hundreds of Outlook emails and retrieve their attachments?

The answer is **Microsoft Graph API**.

---

# Step 2 — Microsoft Graph API

Microsoft Graph API is Microsoft's API for accessing Microsoft 365 services such as Outlook.

It is **not an AWS service**.

Its job in our architecture is to act as the bridge between:

```text
Microsoft Outlook
       ↓
Microsoft Graph API
       ↓
AWS
```

The application can use Graph API to retrieve:

* Email ID
* Sender
* Subject
* Email body
* Attachment information
* Attachment content
* Message metadata

For example:

```text
Outlook Email

From: abc@company.com
Subject: Invoice September

Attachment:
invoice_1023.pdf
```

Graph API retrieves this information so our AWS pipeline can process it.

---

# Step 3 — Send work to Amazon SQS

After reading the emails, we don't want to immediately process hundreds of emails without any buffering.

Instead, we place processing tasks into **Amazon Simple Queue Service (SQS)**.

SQS is a message queue.

Think of it as a waiting line:

```text
Email 1 ─┐
Email 2 ─┤
Email 3 ─┤
Email 4 ─┤
Email 5 ─┤
   ...   ├──→ Amazon SQS
Email 500┘
```

SQS temporarily holds the work until the processing system is ready.

### Why SQS?

It helps us:

* Handle large amounts of incoming work
* Avoid overwhelming Lambda
* Decouple email ingestion from document processing
* Retry failed messages
* Process messages in batches
* Scale processing based on workload

---

# Step 4 — Batch Processing

This is one of the most important parts of the project.

Suppose there are:

```text
500 emails
```

We configure the SQS → Lambda integration with a batch size such as:

```text
10 messages
```

This means Lambda can receive **up to 10 SQS messages in one invocation**.

For example:

```text
Batch 1
Email 1
Email 2
...
Email 10

Batch 2
Email 11
Email 12
...
Email 20

Batch 3
Email 21
...
Email 30
```

We are therefore not designing the system as:

```text
Email 1 → finish
Email 2 → finish
Email 3 → finish
...
Email 500 → finish
```

Instead:

```text
          Amazon SQS
               │
       ┌───────┼───────┐
       ▼       ▼       ▼
     Batch 1 Batch 2 Batch 3
     1–10    11–20    21–30
       │       │       │
       ▼       ▼       ▼
    Lambda  Lambda  Lambda
```

### Important clarification

"Lambda 1", "Lambda 2", and "Lambda 3" in the diagram do **not** mean three different Lambda functions.

We create **one Lambda function**.

AWS can create multiple **concurrent executions** of that same function.

```text
One Lambda Function
        │
        ├── Execution #1 → Batch 1
        ├── Execution #2 → Batch 2
        └── Execution #3 → Batch 3
```

This allows the workload to scale.

---

# Step 5 — AWS Lambda

AWS Lambda is the compute layer.

Our Lambda function receives a batch of SQS messages and processes them.

The Lambda function can:

1. Read the SQS message.
2. Get the email ID.
3. Call Microsoft Graph API.
4. Retrieve the attachment.
5. Download the attachment.
6. Upload the attachment to S3.
7. Store metadata.

Example:

```text
SQS Message
     ↓
Email ID = 12345
     ↓
Graph API
     ↓
Attachment = invoice.pdf
     ↓
Amazon S3
```

---

# Step 6 — Amazon S3

Amazon S3 is the document storage layer.

The attachments are stored in an S3 bucket.

Example:

```text
document-classification-bucket/

├── incoming/
│   ├── invoice_001.pdf
│   ├── resume_002.pdf
│   └── contract_003.pdf
│
├── high-confidence/
│   ├── invoice/
│   ├── resume/
│   ├── contract/
│   ├── purchase-order/
│   └── receipt/
│
└── low-confidence/
    └── manual-review/
```

S3 is responsible for storing the actual files.

DynamoDB, discussed later, stores information **about** those files.

---

# Step 7 — Amazon Textract

Once the documents are stored in S3, they need to be converted into machine-readable text.

This is where **Amazon Textract** is used.

Textract is an AWS managed document-analysis service.

It can extract:

* Printed text
* Words
* Lines
* Forms
* Key-value pairs
* Tables

Example:

### Original document

```text
ABC COMPANY

INVOICE

Invoice Number: INV-1023
Date: 20/09/2026
Amount Due: $2,500
```

### Textract output

```text
ABC COMPANY
INVOICE
Invoice Number: INV-1023
Date: 20/09/2026
Amount Due: $2,500
```

Now the text can be sent to our classification system.

---

# Step 8 — Amazon Bedrock

After Textract extracts the text, we need to determine what type of document it is.

We use **Amazon Bedrock**.

Bedrock is AWS's managed platform for accessing foundation models.

For this project, we can use an appropriate **Amazon Nova** model for classification. For a text-only, speed-oriented classification task, **Nova Micro** is a suitable starting point.

The important advantage is:

```text
No need to train our own ML model
No model server to maintain
No GPU infrastructure
No model deployment pipeline
```

The flow becomes:

```text
Textract
   ↓
Extracted Text
   ↓
Amazon Bedrock
   ↓
Nova
   ↓
Document Category
```

---

# Step 9 — Document Classification

We define categories for our documents.

For example:

```text
Invoice
Resume
Contract
Purchase Order
Receipt
Other
```

We send the extracted text to Bedrock with a classification instruction.

For example:

```text
Classify the following document into exactly one
of these categories:

Invoice
Resume
Contract
Purchase Order
Receipt
Other

Return:
- category
- confidence
- reason
```

Suppose the extracted text contains:

```text
Invoice Number: INV-1001
Amount Due: $4,500
Payment Due Date: 30/09/2026
```

The model may return:

```json
{
  "category": "Invoice",
  "confidence": 96,
  "reason": "The document contains invoice number,
             amount due and payment information."
}
```

---

# Step 10 — Confidence Decision

Now we implement the business rule:

```text
IF confidence >= 90%
       ↓
High Confidence
       ↓
Automatic processing

IF confidence < 90%
       ↓
Low Confidence
       ↓
Manual Review
```

### Example 1

```text
Document: invoice.pdf

Category: Invoice
Confidence: 96%
```

Because:

```text
96 >= 90
```

the document goes to the high-confidence location.

```text
S3
└── high-confidence
    └── invoice
        └── invoice.pdf
```

---

### Example 2

```text
Document: document123.pdf

Category: Contract
Confidence: 72%
```

Because:

```text
72 < 90
```

the document goes to manual review.

```text
S3
└── low-confidence
    └── manual-review
        └── document123.pdf
```

A human can then inspect the document and determine the correct category.

---

# Step 11 — High-Confidence Documents

Our S3 structure can be organized like:

```text
high-confidence/

├── invoice/
│   ├── invoice001.pdf
│   └── invoice002.pdf
│
├── resume/
│   ├── resume001.pdf
│   └── resume002.pdf
│
├── contract/
│   └── contract001.pdf
│
├── purchase-order/
│   └── po001.pdf
│
└── receipt/
    └── receipt001.pdf
```

This means documents that meet the threshold can be automatically organized.

---

# Step 12 — Low-Confidence Documents

Documents below the threshold go to:

```text
low-confidence/

└── manual-review/
    ├── document001.pdf
    ├── document002.pdf
    └── document003.pdf
```

This prevents uncertain classifications from being automatically accepted.

---

# Step 13 — DynamoDB

Finally, we store the processing information in **Amazon DynamoDB**.

S3 stores:

> The actual document.

DynamoDB stores:

> Information about the document and its processing result.

Example record:

```json
{
  "document_id": "DOC-1001",
  "file_name": "invoice001.pdf",
  "category": "Invoice",
  "confidence": 96,
  "status": "AUTO_CLASSIFIED",
  "s3_path": "high-confidence/invoice/invoice001.pdf"
}
```

For a low-confidence document:

```json
{
  "document_id": "DOC-1002",
  "file_name": "document002.pdf",
  "category": "Contract",
  "confidence": 72,
  "status": "MANUAL_REVIEW",
  "s3_path": "low-confidence/manual-review/document002.pdf"
}
```

---

# 🔄 Complete Example

Let's follow **one email** through the entire system.

### Email arrives

```text
Outlook

Subject:
September Invoice

Attachment:
invoice_1023.pdf
```

### 1. Graph API

Graph API reads:

```text
Email ID: 12345
Attachment: invoice_1023.pdf
```

### 2. SQS

A processing message is added:

```text
{
    "email_id": "12345",
    "attachment": "invoice_1023.pdf"
}
```

### 3. Batch

Lambda receives this message along with other messages.

```text
Batch:
Email 1
Email 2
...
Email 10
```

### 4. Lambda

Lambda retrieves the attachment through Graph API.

```text
invoice_1023.pdf
```

### 5. S3

Lambda uploads:

```text
s3://document-classification/incoming/invoice_1023.pdf
```

### 6. Textract

Textract extracts:

```text
Invoice
Invoice Number: INV-1023
Total: $4,500
Due Date: 30/09/2026
```

### 7. Bedrock

Nova classifies:

```text
Category: Invoice
Confidence: 96%
```

### 8. Confidence check

```text
96 >= 90
```

Therefore:

```text
HIGH CONFIDENCE
```

### 9. S3 routing

The document is placed under:

```text
high-confidence/invoice/invoice_1023.pdf
```

### 10. DynamoDB

The system records:

```text
Document ID: DOC-1023
Category: Invoice
Confidence: 96%
Status: AUTO_CLASSIFIED
```

And the complete journey is:

```text
Outlook
   ↓
Graph API
   ↓
SQS
   ↓
Batch
   ↓
Lambda
   ↓
S3
   ↓
Textract
   ↓
Bedrock / Nova
   ↓
Confidence = 96%
   ↓
>= 90%
   ↓
high-confidence/invoice/
   ↓
DynamoDB
```

---

# 🧠 Why Each AWS Component Is Used

| Component           | Role                                                 |
| ------------------- | ---------------------------------------------------- |
| Microsoft Graph API | Connects to Outlook and retrieves emails/attachments |
| Amazon SQS          | Queues incoming processing tasks                     |
| SQS + Lambda        | Enables batch processing                             |
| AWS Lambda          | Executes the processing code                         |
| Amazon S3           | Stores original and classified documents             |
| Amazon Textract     | Extracts text from documents                         |
| Amazon Bedrock      | Provides access to foundation models                 |
| Amazon Nova         | Classifies extracted document text                   |
| Confidence Logic    | Decides automatic vs manual processing               |
| DynamoDB            | Stores classification results and metadata           |

---

# 🚀 Why This Architecture Is Better Than Sequential Processing

Without batching:

```text
Email 1
  ↓
Process
  ↓
Email 2
  ↓
Process
  ↓
Email 3
  ↓
Process
```

With SQS + Lambda:

```text
                 500 Emails
                     ↓
                    SQS
                     ↓
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Batch 1    Batch 2    Batch 3
       1–10       11–20      21–30
          ↓          ↓          ↓
       Lambda     Lambda     Lambda
       execution  execution  execution
          └──────────┼──────────┘
                     ↓
                    S3
                     ↓
                 Textract
                     ↓
                  Bedrock
```

This architecture provides **decoupling, buffering, batching, and concurrent Lambda executions**, which makes it much more suitable for a workload with hundreds of incoming emails.

---


