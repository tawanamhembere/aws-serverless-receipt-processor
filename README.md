# aws-serverless-receipt-processor
# AWS Serverless Receipt Processor

## Architecture

## How It Works

## AWS Services

## Project Structure

## Lambda Processing

## Data Flow

## Security

## Error Handling

## What I Learned

## Future Improvements

## Technologies Used

## How It Works

1. A receipt image or PDF is uploaded to Amazon S3.
2. The S3 upload triggers the Lambda function.
3. Lambda retrieves the bucket and object key from the S3 event.
4. Lambda verifies that the object exists.
5. Lambda sends the document to Amazon Textract.
6. Textract AnalyzeExpense extracts receipt information.
7. Lambda parses the Textract response.
8. A unique receipt ID is generated.
9. The extracted data is stored in DynamoDB.
10. Amazon SES sends a processing notification.
## Data Extracted

Amazon Textract AnalyzeExpense is used to extract:

- Vendor
- Receipt date
- Total amount
- Item name
- Item price
- Item quantity
| Attribute             | Description               |
| --------------------- | ------------------------- |
| `receipt_id`          | Unique receipt identifier |
| `date`                | Receipt date              |
| `vendor`              | Vendor name               |
| `total`               | Receipt total             |
| `items`               | Extracted line items      |
| `s3_path`             | Original receipt location |
| `processed_timestamp` | Processing time           |

## Security

The application uses AWS IAM to control access between services.

The Lambda execution role is intended to follow the principle of least privilege,
allowing the function to access only the AWS resources required for processing.

No AWS access keys, passwords, or secrets are stored in the source code.

## Project Structure

```text
aws-serverless-receipt-processor/
│
├── architecture/
│   └── aws-receipt-processing-architecture.png
│
├── lambda/
│   └── lambda.py
│
├── docs/
│   └── architecture.md
│
├── .gitignore
├── LICENSE
└── README.md
## Future Improvements

- Store receipt totals as numeric DynamoDB attributes
- Add an SQS Dead Letter Queue
- Add retry handling
- Build a web dashboard
- Add expense categorization
- Add authentication
- Add analytics using Amazon QuickSight
- Add automated deployment using AWS SAM or Terraform
- Add automated testing

| AWS Service           | Purpose                          |
| --------------------- | -------------------------------- |
| **Amazon S3**         | Receipt storage and event source |
| **AWS Lambda**        | Serverless processing            |
| **Amazon Textract**   | Receipt data extraction          |
| **Amazon DynamoDB**   | Structured receipt storage       |
| **Amazon SES**        | Email notifications              |
| **AWS IAM**           | Access control                   |
| **Amazon CloudWatch** | Logs and monitoring              |
