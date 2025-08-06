# Understanding the Response Format

This guide explains the data structures and response formats used by the StormParse API.

## API Response Structure

All StormParse API responses follow a consistent format:

```json
{
  "result_type": "SUCCESS" | "ERROR",
  "error": null | "error message",
  "success": { ... response data ... }
}
```

## Upload Response

When you upload a document to `/api/v1/parsing/upload`:

### Request
```python
import requests

with open("document.pdf", 'rb') as f:
    files = {'file': f}
    response = requests.post("https://live-storm-apis-parse-router.sionic.im/api/v1/parsing/upload", files=files)
```

### Response
```json
{
  "result_type": "SUCCESS",
  "error": null,
  "success": {
    "job_id": "30554500-1511-4c81-999f-e3bc7f0fa6fa",
    "state": "REQUESTED",
    "requested_at": "2025-08-05 16:22:31"
  }
}
```

### Fields Explained
- `job_id`: Unique identifier for the parsing job
- `state`: Initial state is always "REQUESTED"
- `requested_at`: Timestamp when the job was created

## Job Status Response

When you check job status at `/api/v1/parsing/job/{job_id}`:

### Processing State
```json
{
  "result_type": "SUCCESS",
  "error": null,
  "success": {
    "job_id": "30554500-1511-4c81-999f-e3bc7f0fa6fa",
    "state": "PROCESSING",
    "requested_at": "2025-08-05 16:22:31",
    "completed_at": null
  }
}
```

### Completed State
```json
{
  "result_type": "SUCCESS",
  "error": null,
  "success": {
    "job_id": "30554500-1511-4c81-999f-e3bc7f0fa6fa",
    "state": "COMPLETED",
    "requested_at": "2025-08-05 16:22:31",
    "completed_at": "2025-08-05 16:23:45",
    "pages": [
      {
        "pageNumber": 1,
        "content": "This is the content of page 1..."
      },
      {
        "pageNumber": 2,
        "content": "This is the content of page 2..."
      }
    ]
  }
}
```

### Job States
- `REQUESTED`: Job created, waiting to start
- `PROCESSING`: Currently being processed
- `COMPLETED`: Successfully completed with results
- `FAILED`: Failed due to error

## Working with Responses

### Accessing Page Content

```python
import requests
import json

# Get job status
response = requests.get(f"https://live-storm-apis-parse-router.sionic.im/api/v1/parsing/job/{job_id}")
result = response.json()

if result['result_type'] == 'SUCCESS' and result['success']['state'] == 'COMPLETED':
    pages = result['success']['pages']
    
    # Iterate through pages
    for page in pages:
        print(f"Page {page['pageNumber']}:")
        print(f"  Character count: {len(page['content'])}")
        print(f"  Word count: {len(page['content'].split())}")
        print(f"  First 100 chars: {page['content'][:100]}...")
```

### Combining Pages

```python
# Get all text as a single string
def combine_pages(pages):
    return "\n\n".join(page['content'] for page in pages)

# Get specific page
def get_page(pages, page_number):
    for page in pages:
        if page['pageNumber'] == page_number:
            return page['content']
    return None

# Get page range
def get_page_range(pages, start, end):
    return [p for p in pages if start <= p['pageNumber'] <= end]
```

### Analyzing Document Structure

```python
def analyze_document(result):
    """Analyze the structure of a parsed document."""
    if result['result_type'] != 'SUCCESS':
        print("Error in response")
        return
    
    data = result['success']
    
    if data['state'] != 'COMPLETED':
        print(f"Job not completed. Current state: {data['state']}")
        return
    
    pages = data['pages']
    
    # Calculate statistics
    total_chars = sum(len(page['content']) for page in pages)
    total_words = sum(len(page['content'].split()) for page in pages)
    
    print(f"Document Analysis:")
    print(f"  Job ID: {data['job_id']}")
    print(f"  Total pages: {len(pages)}")
    print(f"  Total characters: {total_chars:,}")
    print(f"  Total words: {total_words:,}")
    print(f"  Average chars/page: {total_chars // len(pages):,}")
    print(f"  Average words/page: {total_words // len(pages):,}")
    
    # Find empty or nearly empty pages
    empty_pages = [p['pageNumber'] for p in pages if len(p['content'].strip()) < 10]
    if empty_pages:
        print(f"  Empty/sparse pages: {empty_pages}")

# Use the analyzer
response = requests.get(f"https://live-storm-apis-parse-router.sionic.im/api/v1/parsing/job/{job_id}")
analyze_document(response.json())
```

## Error Responses

When errors occur, the API returns an error response:

### Upload Error
```json
{
  "result_type": "ERROR",
  "error": "File type not supported",
  "success": null
}
```

### Job Not Found
```json
{
  "result_type": "ERROR",
  "error": "Job not found",
  "success": null
}
```

### Handling Errors

```python
def handle_api_response(response):
    """Handle API response with error checking."""
    try:
        # Check HTTP status
        if response.status_code != 200:
            print(f"HTTP Error {response.status_code}: {response.text}")
            return None
        
        # Parse JSON
        result = response.json()
        
        # Check result type
        if result['result_type'] != 'SUCCESS':
            print(f"API Error: {result.get('error', 'Unknown error')}")
            return None
        
        return result['success']
        
    except requests.exceptions.JSONDecodeError:
        print("Error: Invalid JSON response")
        return None
    except Exception as e:
        print(f"Error handling response: {e}")
        return None

# Example usage
response = requests.get(f"https://live-storm-apis-parse-router.sionic.im/api/v1/parsing/job/{job_id}")
data = handle_api_response(response)

if data:
    print(f"Job state: {data['state']}")
```

## Advanced Response Processing

### Custom Page Processing

```python
def process_pages(pages, processor_func):
    """Apply a custom processing function to each page."""
    processed = []
    
    for page in pages:
        processed_content = processor_func(page['content'])
        processed.append({
            'pageNumber': page['pageNumber'],
            'content': processed_content,
            'original_length': len(page['content']),
            'processed_length': len(processed_content)
        })
    
    return processed

# Example: Remove extra whitespace
def clean_whitespace(text):
    import re
    # Replace multiple spaces with single space
    text = re.sub(r' +', ' ', text)
    # Replace multiple newlines with double newline
    text = re.sub(r'\n{3,}', '\n\n', text)
    return text.strip()

# Apply processing
cleaned_pages = process_pages(pages, clean_whitespace)
```

### Saving Responses

```python
import json
from datetime import datetime

def save_parsing_result(job_id, result, metadata=None):
    """Save parsing results with metadata."""
    if result['result_type'] != 'SUCCESS' or result['success']['state'] != 'COMPLETED':
        print("Cannot save incomplete or failed job")
        return None
    
    data = result['success']
    
    output = {
        "job_id": job_id,
        "parsed_at": data['completed_at'],
        "saved_at": datetime.now().isoformat(),
        "page_count": len(data['pages']),
        "pages": data['pages'],
        "metadata": metadata or {}
    }
    
    filename = f"parsed_{job_id}.json"
    with open(filename, 'w', encoding='utf-8') as f:
        json.dump(output, f, ensure_ascii=False, indent=2)
    
    print(f"Saved to: {filename}")
    return filename

# Usage
response = requests.get(f"https://live-storm-apis-parse-router.sionic.im/api/v1/parsing/job/{job_id}")
save_parsing_result(job_id, response.json(), {"source": "example.pdf"})
```

## Response Validation

```python
def validate_job_response(response_data):
    """Validate that a job response has expected structure."""
    required_fields = {
        'result_type': str,
        'success': dict
    }
    
    # Check top-level fields
    for field, expected_type in required_fields.items():
        if field not in response_data:
            return False, f"Missing field: {field}"
        if not isinstance(response_data[field], expected_type):
            return False, f"Invalid type for {field}"
    
    # Check success data
    success_data = response_data['success']
    required_success_fields = ['job_id', 'state', 'requested_at']
    
    for field in required_success_fields:
        if field not in success_data:
            return False, f"Missing field in success: {field}"
    
    # Check pages if completed
    if success_data['state'] == 'COMPLETED':
        if 'pages' not in success_data:
            return False, "Completed job missing pages"
        
        if not isinstance(success_data['pages'], list):
            return False, "Pages must be a list"
        
        for i, page in enumerate(success_data['pages']):
            if 'pageNumber' not in page or 'content' not in page:
                return False, f"Page {i} missing required fields"
    
    return True, "Valid response"

# Validate response
response = requests.get(f"https://live-storm-apis-parse-router.sionic.im/api/v1/parsing/job/{job_id}")
is_valid, message = validate_job_response(response.json())
print(f"Validation: {message}")
```

