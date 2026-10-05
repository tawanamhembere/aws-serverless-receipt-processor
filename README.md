# AWS Lambda Receipt Processing System

An automated, serverless document processing pipeline that extracts structured data from uploaded receipts or invoices using AI and stores them in DynamoDB, complete with instant email notifications.

---

## Architecture Overview

```
                          ┌─────────────────┐
                          │   User Upload   │
                          └────────┬────────┘
                                   │
                                   ▼
                          ┌─────────────────┐
                          │   Amazon S3     │
                          │ Receipt Storage │
                          └────────┬────────┘
                                   │ S3 Event Trigger
                                   ▼
                          ┌─────────────────┐
                          │   AWS Lambda    │
                          │ (Python/boto3)  │
                          └───────┬─────────┘
                                  │
         ┌────────────────────────┼────────────────────────┐
         │                        │                        │
         ▼                        ▼                        ▼
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ Amazon Textract  │    │ Amazon DynamoDB  │    │    Amazon SES    │
│ Expense Analysis │    │ Structured Data  │    │ Email Summaries  │
└──────────────────┘    └──────────────────┘    └──────────────────┘
         │                        │                        │
         └────────────────────────┼────────────────────────┘
                                  ▼
                         ┌──────────────────┐
                         │ Amazon CloudWatch│
                         │ Logging & Metrics│
                         └──────────────────┘

```

---

## AWS Services & Cloud Architecture Skills Demonstrated

| AWS Service | Architecture Role | Key Skills & Implementation |
| --- | --- | --- |
| **AWS Lambda** | Compute Engine | Serverless execution, event-driven computing with custom `boto3` handlers, environment configuration, robust error isolation. |
| **Amazon S3** | Object Storage | Event notifications (`s3:ObjectCreated:*`), key URL decoding, metadata lookup using `head_object`. |
| **Amazon Textract** | AI/ML Intelligence | Specialized document analysis using `analyze_expense()` API for complex invoice/receipt key-value parsing. |
| **Amazon DynamoDB** | NoSQL Database | Non-relational schema design, key-value document storage, high-throughput item creation using UUID tracking. |
| **Amazon SES** | Communication | Transactional HTML email notification rendering, dynamic email generation, fault-tolerant execution. |
| **AWS IAM** | Security & Governance | Principle of Least Privilege policy design binding Lambda execution roles to target resource ARNs. |
| **Amazon CloudWatch** | Observability | Centralized logging, operational auditing, and exception diagnosis. |

---

## Features

* **Automated Event-Driven Pipeline:** Zero manual intervention—triggers directly upon S3 object upload.
* **AI Expense Extraction:** Leverages Amazon Textract's domain-tuned `analyze_expense` API to extract line items, prices, vendor, tax, and totals automatically.
* **Resilient URL Handling:** Decodes complex object keys (spaces, unicode characters, and special symbols) dynamically.
* **Non-Blocking Notifications:** Email notifications run in a soft-fail strategy so transient SES delivery issues won't fail the primary data extraction pipeline.
* **Audit Traceability:** Maintains a 1-to-1 dynamic mapping between stored DynamoDB records and source S3 URIs.

---

## Dynamic JSON Schema Output

The Lambda process normalizes unstructured Textract extraction streams into structured JSON objects prior to persistence:

```json
{
  "receipt_id": "c9284f10-18e4-4a22-921b-871d37e6f98a",
  "processed_at": "2026-10-05T10:29:08Z",
  "s3_bucket": "my-app-receipt-storage",
  "s3_key": "uploads/2026/store_receipt_001.pdf",
  "vendor_name": "Tech Hardware Supplies",
  "receipt_date": "2026-10-04",
  "total_amount": "142.50",
  "tax_amount": "12.50",
  "line_items": [
    {
      "item_name": "Mechanical Keyboard",
      "quantity": "1",
      "price": "110.00"
    },
    {
      "item_name": "USB-C Cable 2m",
      "quantity": "2",
      "price": "10.00"
    }
  ]
}

```

---

## Full Lambda Source Code

```python
import os
import json
import uuid
import urllib.parse
from datetime import datetime
import boto3
from botocore.exceptions import ClientError

# 1. AWS Service Client Initialization
s3_client = boto3.client('s3')
textract_client = boto3.client('textract')
dynamodb = boto3.resource('dynamodb')
ses_client = boto3.client('ses')

# Environment Variables
DYNAMODB_TABLE = os.environ.get('DYNAMODB_TABLE', 'Receipts')
SES_SENDER_EMAIL = os.environ.get('SES_SENDER_EMAIL')
SES_RECIPIENT_EMAIL = os.environ.get('SES_RECIPIENT_EMAIL')

table = dynamodb.Table(DYNAMODB_TABLE)


def lambda_handler(event, context):
    """
    Main Lambda entry point triggered by S3 ObjectCreated events.
    """
    print(f"Received event: {json.dumps(event)}")

    try:
        # Extract bucket and key from S3 event
        record = event['Records'][0]['s3']
        bucket_name = record['bucket']['name']
        raw_key = record['object']['key']
        object_key = urllib.parse.unquote_plus(raw_key)

        print(f"Processing object s3://{bucket_name}/{object_key}")

        # Validate object existence in S3
        s3_client.head_object(Bucket=bucket_name, Key=object_key)

        # Process receipt via Amazon Textract
        extracted_data = process_receipt_with_textract(bucket_name, object_key)
        
        # Save structured results in Amazon DynamoDB
        store_receipt_in_dynamodb(extracted_data, bucket_name, object_key)

        # Dispatch email summary via SES
        send_email_notification(extracted_data, bucket_name, object_key)

        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Receipt processed successfully',
                'receipt_id': extracted_data['receipt_id']
            })
        }

    except ClientError as e:
        print(f"AWS ClientError: {e.response['Error']['Message']}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Internal AWS processing error'})
        }
    except Exception as e:
        print(f"Unhandled Exception: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }


def process_receipt_with_textract(bucket, key):
    """
    Analyzes expense document using Textract's analyze_expense API.
    """
    response = textract_client.analyze_expense(
        Document={'S3Object': {'Bucket': bucket, 'Name': key}}
    )

    vendor_name = "UNKNOWN"
    receipt_date = "UNKNOWN"
    total_amount = "0.00"
    items = []

    for expense_doc in response.get('ExpenseDocuments', []):
        # Extract Summary Fields (Vendor, Total, Date)
        for field in expense_doc.get('SummaryFields', []):
            field_type = field.get('Type', {}).get('Text', '')
            field_val = field.get('ValueDetection', {}).get('Text', '')

            if field_type == 'VENDOR_NAME':
                vendor_name = field_val
            elif field_type == 'TOTAL':
                total_amount = field_val
            elif field_type == 'INVOICE_RECEIPT_DATE':
                receipt_date = field_val

        # Extract Line Items
        for line_item_group in expense_doc.get('LineItemGroups', []):
            for line_item in line_item_group.get('LineItems', []):
                item_data = {}
                for field in line_item.get('LineItemExpenseFields', []):
                    f_type = field.get('Type', {}).get('Text', '')
                    f_val = field.get('ValueDetection', {}).get('Text', '')
                    
                    if f_type == 'ITEM':
                        item_data['name'] = f_val
                    elif f_type == 'PRICE':
                        item_data['price'] = f_val
                    elif f_type == 'QUANTITY':
                        item_data['quantity'] = f_val

                if item_data:
                    items.append({
                        'name': item_data.get('name', 'Item'),
                        'price': item_data.get('price', '0.00'),
                        'quantity': item_data.get('quantity', '1')
                    })

    return {
        'receipt_id': str(uuid.uuid4()),
        'vendor': vendor_name,
        'date': receipt_date,
        'total': total_amount,
        'items': items,
        'processed_at': datetime.utcnow().isoformat() + 'Z'
    }


def store_receipt_in_dynamodb(data, bucket, key):
    """
    Persists structured receipt data to Amazon DynamoDB.
    """
    item = {
        'receipt_id': data['receipt_id'],
        'vendor': data['vendor'],
        'date': data['date'],
        'total': data['total'],
        'items': data['items'],
        's3_location': f"s3://{bucket}/{key}",
        'processed_at': data['processed_at']
    }
    table.put_item(Item=item)
    print(f"Persisted item {data['receipt_id']} to DynamoDB table {DYNAMODB_TABLE}")


def send_email_notification(data, bucket, key):
    """
    Generates and sends an HTML notification email using Amazon SES.
    """
    if not SES_SENDER_EMAIL or not SES_RECIPIENT_EMAIL:
        print("SES emails not configured. Skipping email notification.")
        return

    subject = f"Receipt Processed: {data['vendor']} - ${data['total']}"
    
    # Build HTML list of items
    items_html = "".join([
        f"<li>{item['quantity']}x {item['name']} - ${item['price']}</li>"
        for item in data['items']
    ]) or "<li>No individual line items parsed.</li>"

    html_body = f"""
    <html>
    <head></head>
    <body>
        <h2>Receipt Analysis Complete</h2>
        <p><strong>Receipt ID:</strong> {data['receipt_id']}</p>
        <p><strong>Vendor:</strong> {data['vendor']}</p>
        <p><strong>Date:</strong> {data['date']}</p>
        <p><strong>Total Amount:</strong> ${data['total']}</p>
        <p><strong>Source File:</strong> s3://{bucket}/{key}</p>
        <h3>Extracted Line Items</h3>
        <ul>{items_html}</ul>
    </body>
    </html>
    """

    try:
        ses_client.send_email(
            Source=SES_SENDER_EMAIL,
            Destination={'ToAddresses': [SES_RECIPIENT_EMAIL]},
            Message={
                'Subject': {'Data': subject, 'Charset': 'UTF-8'},
                'Body': {'Html': {'Data': html_body, 'Charset': 'UTF-8'}}
            }
        )
        print(f"Sent notification email to {SES_RECIPIENT_EMAIL}")
    except Exception as e:
        # Non-blocking error handling to ensure pipeline success
        print(f"Warning: Failed to send email via SES: {str(e)}")

```

---

## Least-Privilege IAM Execution Policy

Attach this policy to the AWS Lambda Execution Role to enforce security boundaries:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ObjectReadAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:HeadObject"
      ],
      "Resource": "arn:aws:s3:::YOUR_RECEIPT_BUCKET_NAME/*"
    },
    {
      "Sid": "TextractExpenseAnalysis",
      "Effect": "Allow",
      "Action": [
        "textract:AnalyzeExpense"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DynamoDBWriteAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/YOUR_DYNAMODB_TABLE_NAME"
    },
    {
      "Sid": "SESSendEmailAccess",
      "Effect": "Allow",
      "Action": [
        "ses:SendEmail",
        "ses:SendRawEmail"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLogging",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}

```

---

## Getting Started

### Prerequisites

* An active AWS Account.
* AWS CLI configured with administrator or deployer privileges.
* Amazon SES sender and recipient emails verified (or SES in production mode).

### Environment Variables

Configure the following parameters in your Lambda function settings:

| Variable Name | Description | Example Value |
| --- | --- | --- |
| `DYNAMODB_TABLE` | Target DynamoDB table name | `ProcessedReceipts` |
| `SES_SENDER_EMAIL` | Sender address verified in SES | `notifications@yourdomain.com` |
| `SES_RECIPIENT_EMAIL` | Destination email for summaries | `finance@yourdomain.com` |
