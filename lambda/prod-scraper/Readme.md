# 🧠 FAQ Scraper for Bedrock Knowledge Base

This project automates the extraction of FAQ content from a website using Playwright inside an AWS Lambda container. The scraped data is uploaded to S3 in JSON format and ingested by an Amazon Bedrock Knowledge Base.

---

## 🚀 Overview

- **Lambda Function**: Uses a container image (stored in ECR) that runs Playwright to scrape FAQ data.
- **EventBridge Scheduler**: Triggers the Lambda function weekly on **Fridays at 8:00 AM ET** (`cron(0 8 ? * 6 *)`).
- **S3 Bucket**: Stores JSON output in the format:  
  `faq-output/faq_data_YYYYMMDD_HHMMSS.json`.
- **Amazon Bedrock Knowledge Base**: Ingests the data from S3 for natural language querying.

---

## 📁 File Structure

| File | Description |
|------|-------------|
| `lambda_function.py` | Main Lambda handler: orchestrates scraping, formats data, and uploads to S3. |
| `faq_general.py`, `faq_claim.py`, `faq_evidence.py`, `faq_report.py` | Category-specific FAQ extractors. |
| `finder_info.py` | Extracts FAQs with embedded `<a>` tags and formats them as Markdown links. |
| `useful_link.py` | Extracts helpful links and structures them as Q&A pairs. |
| `faq-scraper-stack.yaml` | CloudFormation template to deploy Lambda, S3 bucket, and EventBridge Scheduler. |

---

## 🧩 Extraction Logic

### Supported Categories
- General
- Claiming Property
- Evidence
- Reporting Property

### HTML Assumptions
- Content is inside `<section id="page-content">`
- FAQs structured as:
  - Questions: `<h6 class="card-title">`
  - Answers: `<p class="card-text">`, `<ul>`, `<ol>`

### Process
1. Wait up to 30 seconds for `section#page-content`.
2. Locate all `div.card-body` sections.
3. Extract question from each `<h6.card-title>`.
4. Extract all subsequent content (`<p>`, `<ul>`, `<ol>`) until the next question.
5. Format answers with double line breaks (`\n\n`).
6. Return data as:
   ```json
   [
     {
       "question": "What is unclaimed property?",
       "answer": "Unclaimed property refers to ... "
     },
     ...
   ]
   ```

### Error Handling
- Returns empty list for failed pages.
- Logs timeouts and missing content.

---

## 🔎 Special Extractors

### Fee Finder (`finder_info.py`)
- Same logic as above, but includes `<a>` links.
- Links are converted to Markdown format: `[text](href)`

### Useful Links (`useful_link.py`)
- Extracts all `<a>` tags from `.card-body`.
- Uses link text as the question, Markdown link as the answer.
- Skips invalid or empty links.

---

## ⚙️ Execution Flow

1. **EventBridge Trigger**  
   - Runs every **Friday at 8:00 AM ET**.

2. **Lambda Initialization**  
   - Sets up logging, S3 client, and initializes the scraper.

3. **Scraping Process**
   - Constructs category URLs:  
     `https://mycash.utah.gov/app/<category>`
   - Launches Chromium browser (headless).
   - Navigates to each page and extracts content.

4. **Data Formatting**
   - Aggregates and flattens output to:
     ```json
     [
       {
         "category": "Claiming Property",
         "question": "...",
         "answer": "..."
       },
       ...
     ]
     ```

5. **Upload to S3**
   - Stores JSON at:  
     `s3://<your-bucket>/faq-output/faq_data_YYYYMMDD_HHMMSS.json`

6. **Return Response**
   - Returns FAQ count, S3 path, and extracted content (for debugging).

7. **Cleanup**
   - Gracefully shuts down Playwright browser.

---

## 📦 Deployment Instructions

### 1. Build and Push Docker Image to ECR
```bash
# Build the image
docker build -t faq-scraper .

# Authenticate Docker to ECR
aws ecr get-login-password | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com

# Push the image
docker tag faq-scraper:latest <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/faq-scraper:latest
docker push <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/faq-scraper:latest
```

### 2. Deploy CloudFormation Stack
```bash
aws cloudformation deploy \
  --template-file faq-scraper-stack.yaml \
  --stack-name faq-scraper-stack \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    ImageUri=<ECR_IMAGE_URI> \
    BucketName=<S3_BUCKET_NAME>
```

---

## 🧠 Notes

- Ensure that Lambda has VPC access if needed (for Playwright dependencies).
- Set IAM roles with permissions to access S3 and write logs.
- You can extend the script to support other categories by replicating the logic.

---

## 📬 Contact

If you find this useful or need help integrating it into your project, feel free to open an issue or reach out!