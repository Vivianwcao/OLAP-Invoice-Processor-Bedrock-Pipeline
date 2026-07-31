# Serverless Invoice Pipeline Optimization
## Modernizing Document Processing with AWS Step Functions, Bedrock, and Infrastructure as Code

### Tech Stack & Tools
* **Orchestration & IaC:** AWS Step Functions, AWS SAM (YAML), Docker (`sam build --use-container`), AWS API Gateway, EventBridge, Amazon SNS
* **Compute & Storage:** AWS Lambda (Python 3.13), Amazon S3
* **Parsing & Data Libraries:** PDFPlumber, BeautifulSoup4, Pandas, Regular Expressions (`re`)
* **AI & Vision:** Amazon Bedrock (Nova Lite 2 and Anthropic Claude Sonnet 4.5)
* **Downstream Integration:** OpenInvoice, OpenTicket via platform mTLS APIs

---

## The Alarm: The $500 AWS Bill Spike

In March and April 2026, my manager noticed a sharp increase in our AWS bill. Our monthly Textract charges surged from $120 to over $500, with single days topping $80. He asked me to investigate the cause and bring the costs down.

<img width="800" alt="aws_textract" src="https://github.com/user-attachments/assets/8c4e9f42-e7d9-43be-8ed1-3901970088c8" />

I analyzed our usage and traced the expenses to two invoice pipelines handling AES and CES suppliers. Three main problems caused the cost jump:

* **Problem 1: Scanning 100+ page PDFs for 3 fields:** AES invoices regularly arrive as long PDFs containing over 100 pages of attached tickets. The old system sent the entire file to Textract, costing roughly $10 per invoice just to extract three basic fields like requisition name, AFE number, and purchase date.

* **Problem 2: Invisible retries multiplied costs:** The pipeline was fragmented across AWS and 9 separate [Make (formerly Integromat)](https://www.make.com) scenarios, with 4 for AES and 5 for CES. When files got stuck without error logs, operators hit the retry button to troubleshoot. This re-scanned the 100 page PDF through Textract and added another $10 charge every time.

* **Problem 3: Outgrown low-code tools and zero visibility:** The company originally chose Make during its early startup phase for quick setup. As logic grew complex across 9 scenarios, Make became fragile and dropped files without any error handling or alerts. Clients would email asking where their submissions were, forcing the team to spend hours digging through cross-platform API calls to find where a file vanished. Nobody wanted to touch or maintain the setup, and operators often resorted to blind retries that triggered repeated Textract scans.

*3 Make Scenarios for AES (example)*

<img width="800" alt="AES_make" src="https://github.com/user-attachments/assets/f8f31e51-1e16-47f4-8e3e-610589d10e0e" />

---

## Solution & Results

To resolve this, I redesigned the document processing system natively inside AWS. I consolidated all 9 Make scenarios into an AWS Step Functions state machine, swapped Textract for targeted PDF parsing with Bedrock, and defined the entire stack using AWS SAM templates built in Docker.

### Key Results
* **97% Cost Reduction:** Monthly AWS charges dropped from over $500 down to between $3 and $5 per month, completely eliminating Textract fees.

* **From 5 Minutes to 3 Seconds:** Processing time per invoice dropped from over 5 minutes down to roughly 3 seconds by eliminating cross-platform Make webhooks and keeping all processing native to AWS.

* **Instant Alerts & Effortless Debugging:** Added EventBridge and SNS monitoring rules that send instant email alerts on any pipeline failure. Instead of spending hours hunting through Make scenarios, developers can open Step Functions, view the exact failed state and CloudWatch logs, and fix issues in minutes.

* **Zero Lost Files:** Replaced silent drops with automated retries and explicit failure handling, ensuring every client document is processed reliably without lost transactions.

* **Zero Console Drift:** Defined 100% of the infrastructure in SAM YAML templates built with Docker, making the stack fully reproducible.

*Finished AWS State machine (Step Functions)*

<img width="342" height="702" alt="AESstepfunctions_graph" src="https://github.com/user-attachments/assets/21f0c3bc-c653-4674-b05f-d892a5d634d9" />

---

## Architecture & Data Flow

To keep data consistent as transactions move through the pipeline, every Lambda reads from, enriches, and writes back to a single central JSON state file stored in Amazon S3.

<img width="2720" height="2432" alt="aes_ces_pipeline_detailed" src="https://github.com/user-attachments/assets/8f240544-ec67-4ffb-880c-f3f6f0cd09b1" />


### Data Processing Steps

1. **Payload Ingestion (Lambda Uploader):** Strips raw file bytes from incoming email payloads, uploads the attachments to S3, initializes the central JSON state file in S3, and triggers Step Functions via API Gateway.

2. **Data Parsing (Lambda 1):** Uses BeautifulSoup4 to parse digital text from MHTML files. If an attached CSV is present, Pandas merges it into the parsed structure before writing the updated state JSON back to S3.

3. **Table Detection & Targeted PDF Extraction (Lambda 2):** Slices target table pages from PDF attachments, extracts required fields using PDFPlumber or Amazon Bedrock, and updates the central S3 JSON file.

4. **Conditional Pricebook Mapping (Lambda 3):** Cleans raw CSV pricebooks using Pandas, performs multi-field group matching for contracted AES rates, and saves the final JSON back to S3.

5. **Output Submission:** The pre-existing output Lambda reads the completed JSON file from S3 and submits the data directly to OpenInvoice or OpenTicket via platform mTLS APIs.


---

## Engineering Challenges & Solutions

### 1. Extracting data from 100+ page PDFs without blowing the budget
* **The Challenge:** AES PDFs frequently arrived with over 100 pages, but the required invoice tables were located on only 2 or 3 specific pages. Sending entire files to external tools cost roughly $10 per run.

* **The Solution:** Wrote a custom PDF Slicer that scans headers to locate target tables, slices out only those 2 or 3 relevant pages, and discards the rest. Slicing the file first cut payload sizes and processing time by over 80% before any extraction started.

### 2. Handling shifting table structures and blurry, low-quality scans
* **The Challenge:** Over 90% of incoming PDFs had digitally generated text tables, but row positions and table headers shifted constantly. The remaining small percentage arrived as hand-scanned documents that were rotated, distorted, and hard to read even with human eyes.

* **The Realization:** I tested multiple vision models in Bedrock, including Nova Lite 2, Claude Haiku, Claude Sonnet, and Claude Opus. Throwing more expensive models at blurry scans still resulted in pure OCR errors like confusing 0 with O or 8 with B. That was when I realized LLMs excel at contextual reasoning rather than raw character extraction.

* **The Solution:** Built a two tier strategy. For the 90%+ digital text tables, I used PDFPlumber alongside Pandas parsing logic to adapt to changing header formats with 100% accuracy at zero cost. For the small percentage of scanned files, I pre-fetched valid purchase order numbers and approvers from platform APIs and injected them into the Bedrock prompt. Instead of forcing the model to guess blurry characters, the LLM used reasoning to match extracted text against known valid values.

### 3. Replacing 9 fragile Make scenarios with native AWS orchestration
* **The Challenge:** The old pipeline relied on 9 scattered [Make](https://www.make.com) scenarios that bounced data back and forth between AWS and external webhooks. With zero error handling or alerts, files secretly disappeared. When clients emailed asking where their invoices were, finding the failure meant manually digging through dozens of Make modules and cross-platform API logs. Nobody wanted to touch or maintain the code because fixing one thing often broke another.

* **The Solution:** Systematically audited and tested every Make scenario to map out all hidden business logic, then rebuilt the entire workflow natively in AWS Step Functions using SAM and Docker. Added EventBridge rules and SNS email alerts so any failure triggers an instant notification detailing the exact failed Lambda. Now, when an issue occurs, a developer can open the Step Functions visual console, view the failed state and CloudWatch logs, and debug or re-test the function directly in minutes.
