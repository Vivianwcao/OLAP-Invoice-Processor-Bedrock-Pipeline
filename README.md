# Serverless Invoice Pipeline Optimization
## Modernizing Document Processing with AWS Step Functions, Bedrock, and Infrastructure as Code

### Tech Stack & Tools
* **Orchestration & IaC:** AWS Step Functions, AWS SAM (YAML), Docker (`sam build --use-container`), AWS API Gateway, EventBridge, Amazon SNS
* **Compute & Storage:** AWS Lambda (Python 3.13), Amazon S3
* **Parsing & Data Libraries:** `PDFPlumber`, `BeautifulSoup4`, `Pandas`, `csv`, Regular Expressions (`re`)
* **AI & Vision:** Amazon Bedrock (Nova Lite 2 and Anthropic Claude Sonnet 4.5)
* **Downstream Integration:** OpenInvoice, OpenTicket via platform mTLS APIs

---

## Background & Business Problem
### The $500 AWS Bill Spike

In March and April 2026, my manager noticed a sharp increase in our AWS bill. Our monthly Textract charges went from about $120 to over $500, with some days reaching more than $80. He asked me to find out what was causing the increase and see if I could reduce the costs.

<div align="center">
  <img width="80%" alt="aws_textract" src="https://github.com/user-attachments/assets/8c4e9f42-e7d9-43be-8ed1-3901970088c8" />
</div>

I analyzed our usage and found that most of the cost came from two invoice pipelines for AES and CES suppliers. After digging into the workflows, I identified three main issues that were driving up the costs:

* **Problem 1: Scanning 100+ page PDFs for only 3 fields:** AES invoices regularly arrive as long PDFs containing over 100 pages of attached tickets. The old pipeline sent the entire PDF to Textract just to extract three fields: requisition name, AFE number, and purchase date. This could cost up to $10 for a single invoice.

* **Problem 2: Invisible retries multiplied costs:** The pipeline was split across AWS and 9 separate [Make (formerly Integromat)](https://www.make.com) scenarios, with 4 for AES and 5 for CES. When a file failed without any error logs, operators often retried the workflow to troubleshoot. Each retry sent the same 100 page PDF back through Textract and added another $10 charge.

* **Problem 3: The workflow became difficult to maintainy:** The company originally chose [Make](https://www.make.com) because it was quick to set up during the early startup stage. As more logic was added, the workflow grew into nine separate scenarios that were difficult to follow. There was no error handling or alerting, so files could fail without anyone noticing. When clients asked about missing submissions, the team had to manually trace API calls across multiple systems to find where the file got stuck. Since the workflow was hard to understand, nobody wanted to make changes, and operators often retried failed jobs without knowing the root cause, which triggered repeated Textract scans.

*3 Make Scenarios for AES (example)*

<div align="center">
  <img width="60%" alt="AES_make" src="https://github.com/user-attachments/assets/f8f31e51-1e16-47f4-8e3e-610589d10e0e" />
</div>

---

## Business Solution & Results

To fix these issues, I redesigned the document processing system entirely in AWS. I replaced the nine Make scenarios with a single AWS Step Functions state machine, replaced unnecessary Textract with targeted PDF parsing with `PDFPlumber` and Bedrock, and defined the entire stack using AWS SAM templates built with Docker.

### Key Results
* **97% Cost Reduction:** Reduced monthly AWS costs from over $500 to between $3 and $5 by eliminating Textract charges.

* **Processing Time Reduced from 5 Minutes to 10 Seconds:** Reduced the processing time for each invoice from over 5 minutes to about 10 seconds by replacing cross platform Make webhooks with an AWS Step Functions workflow.

* **Instant Alerts & Effortless Debugging:** Added EventBridge and SNS monitoring to send email alerts whenever a pipeline fails. Developers can open the Step Functions execution, see exactly where the failure occurred, review the CloudWatch logs, and identify the issue within minutes.

* **No Lost Files:** Replaced silent failures with automatic retries and explicit error handling so every client document is either processed successfully or fails with a clear reason.

* **Fully Reproducible Infrastructure:** Defined the entire AWS infrastructure in AWS SAM templates built with Docker, making deployments consistent and repeatable.

*Finished AWS State machine (Step Functions)*

*Route 1 AES*
<img width="1303" height="758" alt="AESstepfunctions_graph" src="https://github.com/user-attachments/assets/910a94ad-7126-4198-9394-3967d31a9f59" />

*Route 2 CES*
<img width="1302" height="762" alt="CESstepfunctions_graph" src="https://github.com/user-attachments/assets/41bcf48f-c9db-4128-9c93-18af5bd5e23b" />
<img width="1302" height="761" alt="AES State machine executions" src="https://github.com/user-attachments/assets/1e77a69e-76a5-4b32-8b7c-1478000a4752" />

---

## Technical Architecture & Data Flow

To keep data consistent as transactions move through the pipeline, every Lambda reads from, enriches, and writes back to a single central JSON state file stored in Amazon S3.

<img width="2720" height="2432" alt="aes_ces_pipeline_detailed" src="https://github.com/user-attachments/assets/8bb1213d-3aba-40a7-960d-c76d994629d5" />

### Data Processing Steps

1. **Payload Ingestion (Lambda Uploader):** Strips raw file bytes from incoming email payloads, uploads the attachments to S3, initializes the central JSON state file in S3, and triggers Step Functions via API Gateway.

2. **Data Parsing (Lambda 1):** Uses BeautifulSoup4 to parse digital text from MHTML files. If an attached CSV is present, uses Python's built-in `csv` library to read and integrate the data into the parsed structure before writing the updated state JSON back to S3.

3. **Table Detection & Targeted PDF Extraction (Lambda 2):** Slices target table pages from PDF attachments, extracts required fields using `PDFPlumber` or Amazon Bedrock, and updates the central S3 JSON file.

4. **Conditional Pricebook Mapping (Lambda 3):** Cleans raw CSV pricebooks using `Pandas`, performs multi-field group matching for contracted AES rates, and saves the final JSON back to S3.

5. **Output Submission:** The pre-existing output Lambda reads the completed JSON file from S3 and submits the data directly to OpenInvoice or OpenTicket via platform mTLS APIs.


---

## Engineering Challenges & Solutions

### 1. Extracting data from 100+ page PDFs without blowing the budget
* **The Challenge:** AES PDFs often contained over 100 pages, but the invoice tables were usually located on only 2 or 3 pages. The old pipeline sent the entire PDF for processing, costing about $10 per invoice just to extract a few fields.

* **The Solution:** Built a custom PDF slicer that scans page headers to find the invoice tables, keeps only the 2 or 3 relevant pages, and ignores the rest. This reduced the file size by over 80% before extraction even started, which significantly lowered both processing time and cost.

### 2. Handling changing table layouts and blurry scanned PDFs
* **The Challenge:** Over 90% of PDFs contained digitally generated text tables, but the row positions and table headers changed frequently. The remaining files were scanned copies that were often rotated or blurry, making them difficult to read even with human eyes.

* **The Realization:** I tested several vision models in Amazon Bedrock, including Nova Lite 2, Claude Haiku, Claude Sonnet, and Claude Opus. Even the more expensive models struggled with blurry scans and often confused characters like 0 and O or 8 and B. That was when I realized LLMs are much better at understanding context than reading unclear text.

* **The Solution:** Built a two tier strategy. For the 90%+ digital text tables, I used `PDFPlumber` with custom Python parsing logic and the built-in `csv` module to handle changing table header formats with 100% accuracy at almost no cost. For the remaining scanned files, I fetched valid purchase order numbers and approvers from platform APIs and included them in the Amazon Bedrock prompt. Instead of guessing unclear characters, the model matched the extracted text against known valid values using context.

### 3. Replacing 9 fragile Make scenarios with AWS Step Functions
* **The Challenge:** The old pipeline relied on nine separate [Make](https://www.make.com) scenarios that passed data between AWS and external webhooks. There was no error handling or alerting, so files could fail without anyone noticing. When a client asked about a missing invoice, we had to trace the workflow across multiple Make scenarios and API logs to find where it failed. The workflow was difficult to maintain, and even small changes could easily cause new problems.

* **The Solution:** I audited and tested every Make scenario to understand the existing business logic, then rebuilt the entire workflow in AWS Step Functions using AWS SAM and Docker. I also added EventBridge rules and SNS email alerts so every pipeline failure sends an automatic notification with the failed Lambda. Now, developers can open the Step Functions visual console, view the failed state and CloudWatch logs, and debug the issue in a few minutes.
