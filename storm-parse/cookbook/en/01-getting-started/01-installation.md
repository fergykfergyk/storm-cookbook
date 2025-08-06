# Installation and Setup

This guide will help you set up your environment to use the StormParse API.

## Prerequisites

- Python 3.7 or higher
- pip package manager
- Access to a StormParse API endpoint

## API Reference
- [https://abit.ly/stormparse-teddynote-api](abit.ly/stormparse-teddynote-api)


## Installation Steps

### 1. Install Required Dependencies

The only required dependency is the `requests` library for making HTTP requests:

```bash
pip install requests
```

### 2. Verify Installation

Create a test script to verify everything is working:

```python
# test_setup.py
import requests

# Test that requests is installed
try:
    import requests
    print("✅ requests library installed successfully!")
    
    # Test API connection (replace with your API URL)
    api_url = "https://live-storm-apis-parse-router.sionic.im"
    try:
        response = requests.get(api_url)
        print(f"✅ Connected to API at {api_url}")
    except requests.exceptions.ConnectionError:
        print(f"⚠️  Could not connect to API at {api_url}")
        print("   Make sure the StormParse API server is running")
        
except ImportError:
    print("❌ requests library not found. Please install with: pip install requests")
```

Run the test:

```bash
python test_setup.py
```

## Configuration

### API Endpoint

You'll need to know your StormParse API endpoint URL. Common configurations:

```python
# Local development
API_URL = "https://live-storm-apis-parse-router.sionic.im"

# Production server (example)
API_URL = "https://api.your-domain.com"

# Using environment variables (recommended)
import os
API_URL = os.getenv("STORMPARSE_API_URL", "https://live-storm-apis-parse-router.sionic.im")
```

### Basic API Test

Test the API endpoints directly:

```python
import requests

# Your API configuration
API_BASE_URL = "https://live-storm-apis-parse-router.sionic.im"
API_KEY = "demo_test"  # Replace with your API key

# Test endpoints
upload_endpoint = f"{API_BASE_URL}/api/v1/parsing/upload"
status_endpoint = f"{API_BASE_URL}/api/v1/parsing/job/{{job_id}}"

print(f"Upload endpoint: {upload_endpoint}")
print(f"Status endpoint: {status_endpoint}")

# Test with authorization
headers = {
    'Authorization': f'Bearer {API_KEY}',
    'Accept': '*/*'
}

# Note: You'll need to include these headers in all API requests
```

## Understanding the API

StormParse uses a simple two-step process:

1. **Upload** - POST your document to `/api/v1/parsing/upload`
   - Returns a `job_id`
   
2. **Check Status & Get Results** - GET from `/api/v1/parsing/job/{job_id}`
   - Returns job status and parsed content when complete

## Troubleshooting

### Common Issues

1. **Connection refused error**
   - Verify the API server is running
   - Check the API URL is correct
   - Ensure no firewall is blocking the connection

2. **requests not found**
   ```bash
   pip install requests
   ```

3. **SSL Certificate errors (for HTTPS endpoints)**
   ```python
   # For development only - not recommended for production
   import requests
   from requests.packages.urllib3.exceptions import InsecureRequestWarning
   requests.packages.urllib3.disable_warnings(InsecureRequestWarning)
   
   # Make requests with verify=False
   response = requests.get(api_url, verify=False)
   ```

## Next Steps

Now that you have the environment set up, proceed to [Your First Parse](./02-first-parse.md) to start parsing documents!