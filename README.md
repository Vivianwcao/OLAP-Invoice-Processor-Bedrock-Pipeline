# Serverless Invoice Pipeline Optimization
## Modernizing Document Processing with AWS Step Functions, Bedrock, and Infrastructure as Code

### Tech Stack & Tools
* **Orchestration & IaC:** AWS Step Functions, AWS SAM (YAML), Docker (`sam build --use-container`), AWS API Gateway, EventBridge, Amazon SNS
* **Compute & Storage:** AWS Lambda (Python 3.13), Amazon S3
* **Parsing & Data Libraries:** `PDFPlumber`, `BeautifulSoup4`, `Pandas`, `csv`, Regular Expressions (`re`)
* **AI & Vision:** Amazon Bedrock (Nova Lite 2 and Anthropic Claude Sonnet 4.5)
* **Downstream Integration:** OpenInvoice, OpenTicket via platform mTLS APIs

---

## The Alarm: The $500 AWS Bill Spike

In March and April 2026, my manager noticed a sharp increase in our AWS bill. Our monthly Textract charges went from about $120 to over $500, with some days reaching more than $80. He asked me to find out what was causing the increase and see if I could reduce the costs.

<img width="800" alt="aws_textract" src="https://github.com/user-attachments/assets/8c4e9f42-e7d9-43be-8ed1-3901970088c8" />

I analyzed our usage and found that most of the cost came from two invoice pipelines for AES and CES suppliers. After digging into the workflows, I identified three main issues that were driving up the costs:

* **Problem 1: Scanning 100+ page PDFs for only 3 fields:** AES invoices regularly arrive as long PDFs containing over 100 pages of attached tickets. The old pipeline sent the entire PDF to Textract just to extract three fields: requisition name, AFE number, and purchase date. This could cost up to $10 for a single invoice.

* **Problem 2: Invisible retries multiplied costs:** The pipeline was split across AWS and 9 separate [Make (formerly Integromat)](https://www.make.com) scenarios, with 4 for AES and 5 for CES. When a file failed without any error logs, operators often retried the workflow to troubleshoot. Each retry sent the same 100 page PDF back through Textract and added another $10 charge.

* **Problem 3: The workflow became difficult to maintainy:** The company originally chose [Make](https://www.make.com) because it was quick to set up during the early startup stage. As more logic was added, the workflow grew into nine separate scenarios that were difficult to follow. There was no error handling or alerting, so files could fail without anyone noticing. When clients asked about missing submissions, the team had to manually trace API calls across multiple systems to find where the file got stuck. Since the workflow was hard to understand, nobody wanted to make changes, and operators often retried failed jobs without knowing the root cause, which triggered repeated Textract scans.

*3 Make Scenarios for AES (example)*

<img width="800" alt="AES_make" src="https://github.com/user-attachments/assets/f8f31e51-1e16-47f4-8e3e-610589d10e0e" />

---

## Solution & Results

To resolve this, I redesigned the document processing system natively inside AWS. I consolidated all 9 Make scenarios into an AWS Step Functions state machine, swapped Textract for targeted PDF parsing with Bedrock, and defined the entire stack using AWS SAM templates built in Docker.

### Key Results
* **97% Cost Reduction:** Monthly AWS charges dropped from over $500 down to between $3 and $5 per month, completely eliminating Textract fees.

* **From 5 Minutes to 3 Seconds:** Processing time per invoice dropped from over 5 minutes down to roughly 3 seconds by eliminating cross-platform Make webhooks and keeping all processing native to AWS.

* **Instant Alerts & Effortless Debugging:** Added EventBridge and SNS monitoring rules that send instant email alerts on any pipeline failure. Instead of manually checking between Make scenarios, developers can open the Step Functions execution, see exactly where it failed, review the CloudWatch logs, and locate the issue in just a few minutes.

* **Zero Lost Files:** Replaced silent drops with automated retries and explicit failure handling, ensuring every client document is processed reliably without lost transactions.

* **Zero Console Drift:** Defined 100% of the infrastructure in SAM YAML templates built with Docker, making the stack fully reproducible.

*Finished AWS State machine (Step Functions)*
<img width="1602" height="677" alt="AESstepfunctions_graph" src="https://github.com/user-attachments/assets/7b1aaef5-5b3a-46f1-a376-5e270a3013c7" />

---

## Architecture & Data Flow

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
* **The Challenge:** AES PDFs frequently arrived with over 100 pages, but the required invoice tables were located on only 2 or 3 specific pages. Sending entire files to external tools cost roughly $10 per run.

* **The Solution:** Wrote a custom PDF Slicer that scans headers to locate target tables, slices out only those 2 or 3 relevant pages, and discards the rest. Slicing the file first cut payload sizes and processing time by over 80% before any extraction started.

### 2. Handling shifting table structures and blurry, low-quality scans
* **The Challenge:** Over 90% of PDFs contained digitally generated text tables, but the row positions and table headers changed frequently. The remaining files were scanned copies, and many were rotated and blurry, making them difficult to read even with human eyes.

* **The Realization:** I tested several vision models in Amazon Bedrock, including Nova Lite 2, Claude Haiku, Sonnet, and Opus. Even the more expensive models struggled with blurry scans and often confused characters like 0 and O or 8 and B. That's when I realized LLMs are much better at understanding context than reading unclear text.

* **The Solution:** Built a two tier strategy. For the 90%+ digital text tables, I used `PDFPlumber` with custom Python parsing logic and the built-in `csv` module to handle changing table header formats with 100% accuracy at ~zero cost. For the small percentage of scanned files, I pre-fetched valid purchase order numbers and approvers from platform APIs and injected them into the Bedrock prompt. Instead of forcing the model to guess blurry characters, the LLM used reasoning to match extracted text against known valid values.

### 3. Replacing 9 fragile Make scenarios with native AWS orchestration
* **The Challenge:** The old pipeline relied on nine separate [Make](https://www.make.com) scenarios that passed data between AWS and external webhooks. There was no error handling or alerting, so files could fail without anyone noticing. When a client asked about a missing invoice, we had to manually check multiple Make scenarios and API logs to figure out what happened. The workflow was difficult to maintain, and even small changes could easily cause new problems.

* **The Solution:** Systematically audited and tested every Make scenario to map out all hidden business logic, then rebuilt the entire workflow natively in AWS Step Functions using SAM and Docker. Added EventBridge rules and SNS email alerts so any failure triggers an instant notification detailing the exact failed Lambda. Now, when an issue occurs, a developer can open the Step Functions visual console, view the failed state and CloudWatch logs, and debug or re-test the function directly in minutes.
