# Serverless Visitor Counter

A live visitor counter for a static website, built with a fully serverless AWS architecture. Every page load triggers an API call that atomically increments a counter in DynamoDB and returns the new total.

**Live site:** http://souhail-portfolio-site.s3-website.us-east-2.amazonaws.com
**API endpoint:** https://bcg1j8q5ai.execute-api.us-east-2.amazonaws.com/count

## Architecture

Visitor's browser
│ (1) loads page from S3
▼
S3 static website
│ (2) JavaScript fetch() on page load
▼
API Gateway (HTTP API, GET /count)
│ (3) invokes
▼
Lambda (Python 3.12)
│ (4) atomic UpdateItem
▼
DynamoDB (visitor-count table)
│ (5) returns new count
▼
Browser displays "Visitor count: N"


## AWS Services Used

| Service | Role |
|---|---|
| **S3** | Static website hosting for the frontend |
| **API Gateway** | HTTP API exposing `GET /count` publicly |
| **Lambda** | Python function that increments and returns the count |
| **DynamoDB** | `visitor-count` table storing the counter (`id = "counter"`) |
| **IAM** | Least-privilege inline policy: Lambda can only `UpdateItem`/`GetItem` on this one table |
| **Budgets** | Zero-spend budget alerting on any charge |

All resources in **us-east-2**, everything within AWS Free Tier limits.

## How It Works

1. The page's JavaScript calls `GET /count` when the site loads.
2. API Gateway routes the request to the Lambda function.
3. Lambda runs an atomic `UpdateItem` with `if_not_exists(count, 0) + 1` — safe under concurrent visitors, no read-before-write race.
4. The new count returns as JSON; the page renders it. CORS headers allow the cross-origin call from S3.

## Project Structure
├── index.html # Site page with the counter fetch snippet
├── lambda_function.py # Lambda handler (boto3 DynamoDB UpdateItem)
└── iam-policy.json # Least-privilege policy for the Lambda execution role

## Lambda Code

See `lambda_function.py`. Key detail: `count` is a DynamoDB reserved word, so the update expression aliases it as `#c`:

```python
table.update_item(
    Key={'id': 'counter'},
    UpdateExpression='SET #c = if_not_exists(#c, :zero) + :inc',
    ExpressionAttributeNames={'#c': 'count'},
    ExpressionAttributeValues={':zero': 0, ':inc': 1},
    ReturnValues='UPDATED_NEW'
)

Troubleshooting Notes (real issues hit during the build)
API returned {"message": "Not Found"} — the request path didn't match any route; the /count suffix was missing from the URL.
Page displayed raw code instead of rendering — S3 served index.html with the wrong Content-Type. Fixed with:
aws s3 cp s3://BUCKET/index.html s3://BUCKET/index.html --content-type "text/html" --metadata-directive REPLACE via CloudShell.
First API was created in the wrong region — API Gateway is regional; resources must live in the same region to connect. Recreated in us-east-2.
IAM policy search returned nothing — skipped the managed policy and wrote a scoped inline policy instead (better practice anyway).
Cost
$0 — Lambda, API Gateway, and DynamoDB all sit comfortably inside Free Tier at this scale, with a zero-spend budget as a safety net.

Future Improvements
Merge the counter into a full portfolio page
Add CloudFront in front of S3 for HTTPS + caching
Request throttling / WAF on the API to prevent counter abuse
