# Your First Document Parse

Let's parse your first document using the StormParse API! This guide will walk you through the basic workflow.

## Basic Parsing Workflow

The StormParse API follows an asynchronous pattern:

1. **Upload** a document → receive a job ID
2. **Poll** the job status until completion
3. **Retrieve** the parsed content

## Example: Parse a Single PDF

### Step 1: Basic Parse

```python
import requests
import time

# Configuration
API_BASE_URL = "https://live-storm-apis-parse-router.sionic.im"
API_KEY = "demo_test"  # Replace with your API key

def parse_document(file_path, api_key=API_KEY, api_base_url=API_BASE_URL):
    """Parse a document using StormParse API."""
    
    # Headers with authorization
    headers = {
        'Authorization': f'Bearer {api_key}',
        'Accept': '*/*'
    }
    
    # Step 1: Upload the document
    upload_url = f"{api_base_url}/api/v1/parsing/upload"
    
    with open(file_path, 'rb') as f:
        files = {'file': f}
        response = requests.post(upload_url, files=files, headers=headers)
    
    if response.status_code != 200:
        print(f"Upload failed: {response.text}")
        return None
    
    result = response.json()
    if result.get('result_type') != 'SUCCESS':
        print(f"Upload error: {result.get('error')}")
        return None
    
    job_id = result['success']['job_id']
    print(f"Document uploaded. Job ID: {job_id}")
    
    # Step 2: Poll for completion
    status_url = f"{api_base_url}/api/v1/parsing/job/{job_id}"
    
    while True:
        response = requests.get(status_url, headers=headers)
        result = response.json()
        
        if result.get('result_type') != 'SUCCESS':
            print(f"Status check error: {result.get('error')}")
            return None
        
        job_data = result['success']
        state = job_data['state']
        
        print(f"Job status: {state}")
        
        if state == 'COMPLETED':
            return job_data['pages']
        elif state == 'FAILED':
            print("Job failed!")
            return None
        
        # Wait before checking again
        time.sleep(2)

# Parse a document
pages = parse_document("sample.pdf")

if pages:
    for page in pages:
        print(f"\n--- Page {page['pageNumber']} ---")
        print(page['content'])
```

### Step 2: Understanding the Process

Let's see what happens behind the scenes:

```python
import requests
import time
import json

API_BASE_URL = "https://live-storm-apis-parse-router.sionic.im"
API_KEY = "demo_test"  # Replace with your API key

# Headers with authorization
headers = {
    'Authorization': f'Bearer {API_KEY}',
    'Accept': '*/*'
}

# Step 1: Upload the document
print("Step 1: Uploading document...")
upload_url = f"{API_BASE_URL}/api/v1/parsing/upload"

with open("sample.pdf", 'rb') as f:
    files = {'file': f}
    response = requests.post(upload_url, files=files, headers=headers)

print(f"Upload response status: {response.status_code}")
upload_result = response.json()
print(f"Upload response: {json.dumps(upload_result, indent=2)}")

# Extract job ID
job_id = upload_result['success']['job_id']
print(f"\nJob ID: {job_id}")

# Step 2: Check job status
print("\nStep 2: Checking job status...")
status_url = f"{API_BASE_URL}/api/v1/parsing/job/{job_id}"

# Poll until complete
max_attempts = 30  # Maximum 60 seconds
attempt = 0

while attempt < max_attempts:
    response = requests.get(status_url, headers=headers)
    status_result = response.json()
    
    state = status_result['success']['state']
    print(f"Attempt {attempt + 1}: Status = {state}")
    
    if state == 'COMPLETED':
        print("\n✅ Parsing completed!")
        pages = status_result['success']['pages']
        print(f"Parsed {len(pages)} pages")
        break
    elif state == 'FAILED':
        print("\n❌ Parsing failed!")
        break
    
    time.sleep(2)
    attempt += 1
```

## Working with Different File Types

StormParse supports various document formats:

```python
def parse_any_document(file_path):
    """Parse any supported document type."""
    
    # Determine file type
    file_extension = file_path.split('.')[-1].lower()
    supported_types = ['pdf', 'docx', 'doc', 'txt']
    
    if file_extension not in supported_types:
        print(f"Unsupported file type: {file_extension}")
        return None
    
    print(f"Parsing {file_extension.upper()} file: {file_path}")
    
    # Use the same parsing process
    return parse_document(file_path)

# Examples
pdf_pages = parse_any_document("report.pdf")
docx_pages = parse_any_document("contract.docx")
txt_pages = parse_any_document("notes.txt")
```

## Handling Large Documents

For large documents that take longer to process:

```python
def parse_large_document(file_path, api_key, timeout=600, poll_interval=5):
    """Parse large documents with custom timeout and polling interval."""
    
    API_BASE_URL = "https://live-storm-apis-parse-router.sionic.im"
    
    # Headers with authorization
    headers = {
        'Authorization': f'Bearer {api_key}',
        'Accept': '*/*'
    }
    
    # Upload
    with open(file_path, 'rb') as f:
        files = {'file': f}
        response = requests.post(f"{API_BASE_URL}/api/v1/parsing/upload", files=files, headers=headers)
    
    result = response.json()
    job_id = result['success']['job_id']
    
    # Poll with custom settings
    start_time = time.time()
    
    while time.time() - start_time < timeout:
        response = requests.get(f"{API_BASE_URL}/api/v1/parsing/job/{job_id}", headers=headers)
        result = response.json()
        
        state = result['success']['state']
        
        if state == 'COMPLETED':
            return result['success']['pages']
        elif state == 'FAILED':
            raise Exception("Parsing failed")
        
        time.sleep(poll_interval)
    
    raise Exception(f"Timeout after {timeout} seconds")

# Parse a large document
large_document = "annual_report_500_pages.pdf"
pages = parse_large_document(large_document, timeout=600, poll_interval=5)
```

## Error Handling

Always include error handling in production code:

```python
def safe_parse_document(file_path):
    """Parse document with comprehensive error handling."""
    
    try:
        # Check if file exists
        import os
        if not os.path.exists(file_path):
            print(f"Error: File not found: {file_path}")
            return None
        
        # Check file size
        file_size = os.path.getsize(file_path)
        if file_size > 100 * 1024 * 1024:  # 100MB
            print(f"Warning: Large file ({file_size / 1024 / 1024:.1f}MB)")
        
        # Parse document
        pages = parse_document(file_path)
        
        if pages:
            print(f"✅ Successfully parsed {len(pages)} pages")
            return pages
        else:
            print("❌ Parsing failed")
            return None
            
    except requests.exceptions.ConnectionError:
        print("Error: Cannot connect to StormParse API")
    except requests.exceptions.Timeout:
        print("Error: Request timed out")
    except Exception as e:
        print(f"Unexpected error: {e}")
    
    return None

# Safe parsing
result = safe_parse_document("document.pdf")
```

## Complete Example Script

Here's a complete script that parses a document with proper error handling:

```python
#!/usr/bin/env python3
"""
parse_document.py - Parse a document using StormParse API
"""

import sys
import os
import requests
import time
import json

def parse_and_save(file_path, api_key, api_url="https://live-storm-apis-parse-router.sionic.im", save_output=True):
    """Parse a document and optionally save the output."""
    
    print(f"Parsing document: {file_path}")
    
    # Check file
    if not os.path.exists(file_path):
        print(f"Error: File not found: {file_path}")
        return False
    
    try:
        # Upload document
        print("Uploading...")
        with open(file_path, 'rb') as f:
            files = {'file': f}
            response = requests.post(f"{api_url}/api/v1/parsing/upload", files=files)
        
        if response.status_code != 200:
            print(f"Upload failed with status {response.status_code}")
            return False
        
        result = response.json()
        if result.get('result_type') != 'SUCCESS':
            print(f"Upload error: {result.get('error')}")
            return False
        
        job_id = result['success']['job_id']
        print(f"Job ID: {job_id}")
        
        # Poll for completion
        print("Processing", end="")
        status_url = f"{api_url}/api/v1/parsing/job/{job_id}"
        
        for _ in range(150):  # Max 5 minutes
            response = requests.get(status_url)
            result = response.json()
            
            if result.get('result_type') != 'SUCCESS':
                print(f"\nStatus error: {result.get('error')}")
                return False
            
            state = result['success']['state']
            
            if state == 'COMPLETED':
                print("\n✅ Completed!")
                pages = result['success']['pages']
                
                # Display summary
                total_chars = sum(len(p['content']) for p in pages)
                print(f"Parsed {len(pages)} pages, {total_chars:,} characters")
                
                # Save output if requested
                if save_output:
                    output_file = file_path.rsplit('.', 1)[0] + '_parsed.txt'
                    with open(output_file, 'w', encoding='utf-8') as f:
                        for page in pages:
                            f.write(f"--- Page {page['pageNumber']} ---\n")
                            f.write(page['content'])
                            f.write("\n\n")
                    print(f"Saved to: {output_file}")
                
                return True
            
            elif state == 'FAILED':
                print("\n❌ Parsing failed!")
                return False
            
            print(".", end="", flush=True)
            time.sleep(2)
        
        print("\n⏱️  Timeout!")
        return False
        
    except Exception as e:
        print(f"\nError: {e}")
        return False

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python parse_document.py <file_path> [api_url]")
        sys.exit(1)
    
    file_path = sys.argv[1]
    api_url = sys.argv[2] if len(sys.argv) > 2 else "https://live-storm-apis-parse-router.sionic.im"
    
    success = parse_and_save(file_path, api_url)
    sys.exit(0 if success else 1)
```

Run the script:

```bash
python parse_document.py sample.pdf
python parse_document.py document.docx https://api.example.com
```

## Next Steps

Now that you've successfully parsed your first document, explore:

- [Understanding Response Format](./03-response-format.md) - Learn about the data structure
- [PDF Parsing Examples](../02-common-use-cases/01-pdf-parsing.md) - Real-world examples
- [Interactive Tutorial](../02-common-use-cases/pdf_parsing_tutorial.ipynb) - Jupyter notebook tutorial