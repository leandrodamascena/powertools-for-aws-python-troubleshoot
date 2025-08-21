# Powertools for AWS Lambda (Python)A - Troubleshooting Project

This is a SAM (Serverless Application Model) project designed for troubleshooting and testing Powertools for AWS Lambda (Pyton). The project includes a Lambda function connected to API Gateway for easy deployment and testing.

## Project Structure

- `src/` - Contains the Lambda function code and dependencies
  - `src/app.py` - Main Lambda function handler
  - `src/requirements.txt` - Python dependencies
- `events/` - Sample invocation events for testing
- `template.yaml` - SAM template defining AWS resources (Lambda + API Gateway)

## Prerequisites

Before getting started, ensure you have the following installed:

- [SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html)
- [Python 3.9+](https://www.python.org/downloads/)
- [Docker](https://hub.docker.com/search/?type=edition&offering=community)
- AWS CLI configured with appropriate credentials

## Modifying Dependencies

### Adding Powertools for AWS Lambda

To add Powertools for AWS Lambda to your project, update the `src/requirements.txt` file:

```txt
requests
aws-lambda-powertools
```

For specific Powertools features, you can add:

```txt
requests
aws-lambda-powertools[all]
aws-lambda-powertools[tracer]
aws-lambda-powertools[logger]
aws-lambda-powertools[metrics]
aws-lambda-powertools[parser]
```

### Common Powertools Dependencies

```txt
# Core Powertools
aws-lambda-powertools

# Additional useful libraries for troubleshooting
boto3
botocore
pydantic
```

## Modifying the Lambda Function

### Basic Powertools Integration

Update `src/app.py` to include Powertools features:

```python
import json
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.logging import correlation_paths
from aws_lambda_powertools.metrics import MetricUnit

# Initialize Powertools
logger = Logger()
tracer = Tracer()
metrics = Metrics()

@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
@tracer.capture_lambda_handler
@metrics.log_metrics
def lambda_handler(event, context):
    logger.info("Lambda function invoked", extra={"event": event})
    
    # Add custom metrics
    metrics.add_metric(name="InvocationCount", unit=MetricUnit.Count, value=1)
    
    try:
        # Your business logic here
        result = {"message": "Hello from Powertools!", "statusCode": 200}
        
        logger.info("Function executed successfully", extra={"result": result})
        return {
            "statusCode": 200,
            "body": json.dumps(result),
            "headers": {
                "Content-Type": "application/json"
            }
        }
    except Exception as e:
        logger.exception("Error processing request")
        tracer.put_annotation(key="error", value=str(e))
        
        return {
            "statusCode": 500,
            "body": json.dumps({"error": "Internal server error"}),
            "headers": {
                "Content-Type": "application/json"
            }
        }
```

## Local Development and Testing

### Building the Application

Build your application using the SAM CLI:

```bash
sam build --use-container
```

This command installs dependencies from `src/requirements.txt` and creates a deployment package.

### Starting the Local API

The template is already configured to connect your Lambda function to API Gateway. Start the local API server:

```bash
sam local start-api
```

This starts a local API Gateway on `http://localhost:3000`. The function will be available at any path since the template uses `/{proxy+}` with `ANY` method.

### Testing Locally

#### Using curl

```bash
# Test with GET request
curl http://localhost:3000/

# Test with POST request
curl -X POST http://localhost:3000/test \
  -H "Content-Type: application/json" \
  -d '{"key": "value"}'

# Test with query parameters
curl "http://localhost:3000/hello?name=World"
```

#### Using the SAM CLI invoke command

Test the function directly with a test event:

```bash
sam local invoke HelloWorldFunction --event events/event.json
```

#### Custom Test Events

Create custom test events in the `events/` folder:

```json
{
  "httpMethod": "GET",
  "path": "/test",
  "queryStringParameters": {
    "param1": "value1"
  },
  "headers": {
    "Content-Type": "application/json"
  },
  "body": "{\"test\": \"data\"}"
}
```

## Event Examples

The `events/` folder contains a comprehensive collection of sample events that integrate with AWS Lambda, covering most AWS services that can trigger Lambda functions. These events are useful for local testing and understanding the structure of different event sources.

### Available Event Types

The folder includes events for:
- **API Gateway** (REST API, HTTP API, WebSocket)
- **S3** (Object events, EventBridge notifications)
- **DynamoDB** (Streams, tumbling windows)
- **SQS** (Standard, FIFO, DLQ triggers)
- **SNS** (Direct and SQS integration)
- **Kinesis** (Data Streams, Firehose)
- **EventBridge** (Custom events, scheduled events)
- **Cognito** (Authentication triggers)
- **CloudWatch** (Logs, alarms, dashboards)
- **AppSync** (Resolvers, authorizers)
- **And many more...**

### Using Event Examples

Test your function with different event types:

```bash
# Test with API Gateway event
sam local invoke HelloWorldFunction --event events/apiGatewayProxyEvent.json

# Test with S3 event
sam local invoke HelloWorldFunction --event events/s3Event.json

# Test with SQS event
sam local invoke HelloWorldFunction --event events/sqsEvent.json
```

This allows you to test how your Powertools-enabled function handles different AWS service integrations without deploying to AWS.

### Environment Variables for Powertools

Set environment variables for local testing:

```bash
# Enable debug logging
export POWERTOOLS_LOG_LEVEL=DEBUG

# Set service name for tracing
export POWERTOOLS_SERVICE_NAME=troubleshoot-service

# Start local API with environment variables
sam local start-api --env-vars env.json
```

Create an `env.json` file:

```json
{
  "HelloWorldFunction": {
    "POWERTOOLS_LOG_LEVEL": "DEBUG",
    "POWERTOOLS_SERVICE_NAME": "troubleshoot-service",
    "POWERTOOLS_METRICS_NAMESPACE": "TroubleshootApp"
  }
}
```

## Deployment

### First Time Deployment

Deploy your application to AWS:

```bash
sam build --use-container
sam deploy --guided
```

Follow the prompts to configure:
- **Stack Name**: Choose a unique name (e.g., `powertools-troubleshoot`)
- **AWS Region**: Your preferred region
- **Confirm changes**: Yes (recommended for first deployment)
- **Allow IAM role creation**: Yes
- **Save parameters**: Yes

### Subsequent Deployments

After the first deployment, simply run:

```bash
sam build --use-container && sam deploy
```

The API Gateway endpoint URL will be displayed in the output after deployment.

## Troubleshooting Common Issues

### Powertools Import Errors

If you encounter import errors:

```bash
# Rebuild with container to ensure proper dependencies
sam build --use-container

# Check if powertools is in requirements.txt
cat src/requirements.txt
```

### Local Testing Issues

```bash
# Check if Docker is running
docker ps

# Restart local API if needed
sam local start-api --debug

# Test with verbose output
sam local invoke HelloWorldFunction --event events/event.json --debug
```

### Logging and Debugging

Enable debug logging in your function:

```python
import os
os.environ['POWERTOOLS_LOG_LEVEL'] = 'DEBUG'
```

### Memory and Timeout Issues

Update the `template.yaml` if you need more resources:

```yaml
Globals:
  Function:
    Timeout: 30
    MemorySize: 512  # Add this line
```

## Build Issues and Multi-Architecture Support

### Architecture Compatibility Problems

Lambda supports both x86_64 and arm64 architectures. Build issues often occur when dependencies are compiled for the wrong architecture.

#### Common Architecture Issues

```bash
# Error: ImportError: /var/task/lib/python3.x/site-packages/some_package.so: cannot open shared object file
# This usually indicates architecture mismatch
```

#### Solutions for Architecture Issues

1. **Use SAM with Container Builds** (Recommended):
```bash
# Always use container builds for consistent architecture
sam build --use-container
```

2. **Specify Architecture in Template**:
```yaml
Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      Architectures:
        - x86_64  # or arm64
      # ... other properties
```

3. **Force Architecture in Docker Build**:
```bash
# For ARM64 Lambda functions
sam build --use-container --build-image public.ecr.aws/sam/build-python3.9:latest-arm64

# For x86_64 Lambda functions  
sam build --use-container --build-image public.ecr.aws/sam/build-python3.9:latest-x86_64
```

### Build Tool Issues (pip, uv, poetry)

#### pip Issues

Common pip problems and solutions:

```bash
# Problem: pip install fails with compilation errors
# Solution: Use pre-compiled wheels or container builds
sam build --use-container

# Problem: Different pip versions causing issues
# Solution: Pin pip version in requirements.txt
pip==23.3.1
aws-lambda-powertools
```

#### uv Build Tool

If using `uv` as a faster pip replacement:

```bash
# Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# Use uv in your build process
uv pip install -r requirements.txt --target ./package
```

Create a custom build script with uv:

```bash
#!/bin/bash
# build.sh
set -e

echo "Building with uv..."
uv pip install -r src/requirements.txt --target .aws-sam/build/HelloWorldFunction/

# Copy source code
cp -r src/* .aws-sam/build/HelloWorldFunction/
```

#### Poetry Integration

For projects using Poetry:

```toml
# pyproject.toml
[tool.poetry.dependencies]
python = "^3.9"
aws-lambda-powertools = "^2.0.0"
requests = "^2.31.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

Build with Poetry:

```bash
# Export requirements for SAM
poetry export -f requirements.txt --output src/requirements.txt --without-hashes

# Then build normally
sam build --use-container
```

### Native Dependencies and Compilation Issues

#### C Extensions and Native Libraries

Some Python packages require compilation:

```txt
# Problematic packages that need compilation
numpy
pandas
Pillow
cryptography
```

Solutions:

1. **Use Lambda Layers**:
```yaml
# template.yaml
Resources:
  PowertoolsLayer:
    Type: AWS::Serverless::LayerVersion
    Properties:
      LayerName: powertools-layer
      ContentUri: layers/powertools/
      CompatibleRuntimes:
        - python3.9
        - python3.10
      CompatibleArchitectures:
        - x86_64

  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      Layers:
        - !Ref PowertoolsLayer
```

2. **Use Pre-built Layers**:
```yaml
# Use AWS-provided Powertools layer
Layers:
  - arn:aws:lambda:us-east-1:017000801446:layer:AWSLambdaPowertoolsPythonV3:21
```

3. **Container Images** (for complex dependencies):
```dockerfile
# Dockerfile
FROM public.ecr.aws/lambda/python:3.9

COPY requirements.txt ${LAMBDA_TASK_ROOT}
RUN pip install -r requirements.txt

COPY src/ ${LAMBDA_TASK_ROOT}

CMD ["app.lambda_handler"]
```

Update template for container:
```yaml
Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Properties:
      PackageType: Image
      ImageUri: your-account.dkr.ecr.region.amazonaws.com/your-repo:latest
```

### Platform-Specific Build Issues

#### macOS Silicon (M1/M2) Issues

```bash
# Problem: Building on Apple Silicon for x86_64 Lambda
# Solution: Use Docker with platform specification
docker run --platform manylinux2014_x86_64 -v $(pwd):/workspace -w /workspace python:3.9 pip install -r requirements.txt -t ./package

# Or use SAM with specific architecture
sam build --use-container --build-image public.ecr.aws/sam/build-python3.9:latest-x86_64
```

#### Windows Build Issues

```bash
# Problem: Path separators and line endings
# Solution: Use WSL2 or Docker Desktop with Linux containers
sam build --use-container

# Ensure requirements.txt has Unix line endings
dos2unix src/requirements.txt
```

### Debugging Build Problems

#### Enable Verbose Build Output

```bash
# Debug SAM build issues
sam build --use-container --debug

# Check build artifacts
ls -la .aws-sam/build/HelloWorldFunction/
```

#### Common Build Error Solutions

```bash
# Error: "No module named '_ctypes'"
# Solution: Use container build or update base image

# Error: "Microsoft Visual C++ 14.0 is required"
# Solution: Use Linux container build on Windows

# Error: "Failed building wheel for package"
# Solution: Install build dependencies or use pre-compiled wheels
pip install --only-binary=all package-name
```

#### Build Performance Optimization

```bash
# Use build cache
export SAM_CLI_TELEMETRY=0
sam build --use-container --cached

# Parallel builds
sam build --use-container --parallel

# Skip dependency resolution if unchanged
sam build --use-container --skip-pull-image
```

## API Gateway Configuration

The template uses a catch-all configuration that routes all requests to your Lambda function:

```yaml
Events:
  HelloWorld:
    Type: Api
    Properties:
      Path: /{proxy+}
      Method: ANY
```

This means your function will receive all HTTP methods (GET, POST, PUT, DELETE, etc.) on any path. Handle different routes in your Lambda function code:

```python
def lambda_handler(event, context):
    http_method = event.get('httpMethod')
    path = event.get('path')
    
    if http_method == 'GET' and path == '/health':
        return health_check()
    elif http_method == 'POST' and path == '/process':
        return process_data(event)
    else:
        return {
            "statusCode": 404,
            "body": json.dumps({"error": "Not found"})
        }
```

## Monitoring and Logs

### Viewing Lambda Logs

After depl, view your fun

```bash
# TTE`:logs in real-time
sam logs -n HelloWorldFunction --stack-name your-stack-name --tail

# View logs from the last 10 minutes
sam logs -n HelloWorldFunction --stack-name your-stack-name --start-time '10min ago'

# Filter logs by keyword
sam logs -n HelloWorldFunction --stack-name your-stack-name --filter ERROR
```

### Powertools Structured Logging

With Powertools Logger, your logs will be structured JSON:

```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "INFO",
  "message": "Lambda function invoked",
  "service": "troubleshoot-service",
  "correlation_id": "abc-123-def",
  "event": {...}
}
```

### X-Ray Tracing

Enable X-Ray tracing in your template:

```yaml
Globals:
  Function:
    Tracing: Active
```

## Example Powertools Implementations

### Logger with Correlation IDs

```python
from aws_lambda_powertools import Logger
from aws_lambda_powertools.logging import correlation_paths

logger = Logger()

@logger.inject_lambda_context(correlation_id_path=correlation_paths.API_GATEWAY_REST)
def lambda_handler(event, context):
    logger.info("Processing request", extra={"user_id": "123"})
    return {"statusCode": 200, "body": "Success"}
```

### Metrics Collection

```python
from aws_lambda_powertools import Metrics
from aws_lambda_powertools.metrics import MetricUnit

metrics = Metrics(namespace="TroubleshootApp")

@metrics.log_metrics
def lambda_handler(event, context):
    metrics.add_metric(name="RequestCount", unit=MetricUnit.Count, value=1)
    metrics.add_metadata(key="path", value=event.get("path"))
    return {"statusCode": 200}
```

### Request Validation with Parser

```python
from aws_lambda_powertools.utilities.parser import parse
from aws_lambda_powertools.utilities.parser.models import APIGatewayProxyEvent
from pydantic import BaseModel

class UserRequest(BaseModel):
    name: str
    email: str

@parse(model=APIGatewayProxyEvent)
def lambda_handler(event: APIGatewayProxyEvent, context):
    # Parse request body
    user_data = UserRequest.parse_raw(event.body)
    logger.info("User data received", extra={"user": user_data.dict()})
    return {"statusCode": 200}
```

## Cleanup

To delete the deployed application:

```bash
sam delete --stack-name your-stack-name
```

## Useful Resources

- [Powertools for AWS Lambda documentation](httpsrtools.aws.dev/lambda/python/)
- [SAM CLI Documentation](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html)
- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [API Gateway Lambda Integration](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-integrations.html)
