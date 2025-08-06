# PDF Document Parsing with StormParse

This guide demonstrates how to parse PDF documents using StormParse with practical examples.

## Quick Start

```python
import requests
import time

# Configuration
API_BASE_URL = "https://live-storm-apis-parse-router.sionic.im"
API_KEY = "demo_test"  # Replace with your API key

# Headers with authorization
headers = {
    'Authorization': f'Bearer {API_KEY}',
    'Accept': '*/*'
}

# Upload and parse a PDF
with open("example.pdf", 'rb') as f:
    files = {'file': f}
    response = requests.post(f"{API_BASE_URL}/api/v1/parsing/upload", files=files, headers=headers)

job_id = response.json()['success']['job_id']

# Wait for completion and get results
while True:
    response = requests.get(f"{API_BASE_URL}/api/v1/parsing/job/{job_id}", headers=headers)
    data = response.json()
    
    if data['success']['state'] == 'COMPLETED':
        pages = data['success']['pages']
        break
    elif data['success']['state'] == 'FAILED':
        print("Parsing failed")
        break
    
    time.sleep(2)

# Display extracted text
for page in pages:
    print(f"Page {page['pageNumber']}:")
    print(page['content'])
    print("-" * 50)
```

## Example 1: Basic PDF Parsing

Parse a simple PDF and extract all text content.

```python
import requests
import time

def parse_simple_pdf(file_path, api_key, api_base_url="https://live-storm-apis-parse-router.sionic.im"):
    """Parse a PDF and return extracted text."""
    
    # Headers with authorization
    headers = {
        'Authorization': f'Bearer {api_key}',
        'Accept': '*/*'
    }
    
    try:
        # Step 1: Upload document
        with open(file_path, 'rb') as f:
            files = {'file': f}
            response = requests.post(f"{api_base_url}/api/v1/parsing/upload", files=files, headers=headers)
        
        response.raise_for_status()
        result = response.json()
        
        if result['result_type'] != 'SUCCESS':
            raise Exception(f"Upload failed: {result.get('error')}")
        
        job_id = result['success']['job_id']
        print(f"Job ID: {job_id}")
        
        # Step 2: Wait for completion
        start_time = time.time()
        timeout = 300  # 5 minutes
        
        while time.time() - start_time < timeout:
            response = requests.get(f"{api_base_url}/api/v1/parsing/job/{job_id}", headers=headers)
            response.raise_for_status()
            result = response.json()
            
            if result['result_type'] != 'SUCCESS':
                raise Exception(f"Status check failed: {result.get('error')}")
            
            state = result['success']['state']
            
            if state == 'COMPLETED':
                # Extract text from all pages
                pages = result['success']['pages']
                full_text = "\n\n".join(page['content'] for page in pages)
                
                print(f"Successfully parsed {len(pages)} pages")
                print(f"Total characters: {len(full_text)}")
                
                return full_text
                
            elif state == 'FAILED':
                raise Exception("Document parsing failed")
            
            # Still processing
            time.sleep(2)
        
        raise Exception("Parsing timeout")
        
    except Exception as e:
        print(f"Error parsing PDF: {e}")
        return None

# Example usage with actual PDF files
API_KEY = "demo_test"  # Replace with your API key
text = parse_simple_pdf("../99_example_docs/삼성 재무제표.pdf", API_KEY)
```

## Running the Examples

To run these examples:

1. Make sure you have Python 3.7+ and the `requests` library installed
2. Replace `API_KEY` with your actual StormParse API key
3. Place your PDF files in the `99_example_docs` directory
4. Run the examples using Python

For a complete interactive tutorial with more examples, see the [PDF Parsing Tutorial Notebook](./01-pdf_parsing_tutorial.ipynb).

## Troubleshooting

### Common Issues

1. **Authentication Error**
   - Make sure your API key is valid
   - Check that the Authorization header is properly formatted

2. **Large PDF files timing out**
   ```python
   # Increase timeout for large files
   timeout = 600  # 10 minutes
   ```

3. **Connection errors**
   - Verify the API URL is correct
   - Check your internet connection
   - Ensure the API service is running