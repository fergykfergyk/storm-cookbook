# Authentication

Storm API uses API key authentication passed in request headers.

## Authentication Method

### Header-Based Authentication

Include your API key in the `storm-api-key` header:

```python
headers = {
    "storm-api-key": "your-api-key-here"
}
```

## Complete Example

```python
import requests

class StormClient:
    def __init__(self, api_key, base_url="https://https://live-stargate.sionic.im"):
        self.api_key = api_key
        self.base_url = base_url
        self.headers = {"storm-api-key": api_key}
    
    def get(self, endpoint, params=None):
        """GET request with authentication."""
        url = f"{self.base_url}{endpoint}"
        response = requests.get(url, headers=self.headers, params=params)
        response.raise_for_status()
        return response.json()
    
    def post(self, endpoint, json=None, data=None, files=None):
        """POST request with authentication."""
        url = f"{self.base_url}{endpoint}"
        response = requests.post(
            url, 
            headers=self.headers, 
            json=json, 
            data=data, 
            files=files
        )
        response.raise_for_status()
        return response.json()

# Initialize client
client = StormClient(api_key="your-api-key")

# Test authentication
agents = client.get("/api/v2/agents")
print(f"Authentication successful! Found {len(agents['data']['data'])} agents.")
```

## Security Best Practices

### 1. Never Hard-code API Keys

❌ **Bad:**
```python
API_KEY = "sk_live_abcd1234"  # Never do this!
```

✅ **Good:**
```python
import os
API_KEY = os.environ.get("STORM_API_KEY")
```

### 2. Use Environment Variables

```bash
# .bashrc or .zshrc
export STORM_API_KEY="your-api-key-here"
```

### 3. Secure Storage

For production applications:

```python
import keyring

# Store API key securely
keyring.set_password("storm-api", "api-key", "your-key")

# Retrieve API key
api_key = keyring.get_password("storm-api", "api-key")
```

## Error Handling

Handle authentication errors gracefully:

```python
def make_authenticated_request(endpoint, api_key):
    """Make request with proper error handling."""
    try:
        response = requests.get(
            f"https://https://live-stargate.sionic.im{endpoint}",
            headers={"storm-api-key": api_key}
        )
        
        if response.status_code == 401:
            print("❌ Authentication failed. Check your API key.")
            return None
        elif response.status_code == 403:
            print("❌ Access forbidden. Check your permissions.")
            return None
        
        response.raise_for_status()
        return response.json()
        
    except requests.exceptions.RequestException as e:
        print(f"❌ Request failed: {e}")
        return None
```

## Rate Limiting

Storm API implements rate limiting. Handle it properly:

```python
import time

def request_with_retry(client, method, endpoint, max_retries=3, **kwargs):
    """Make request with automatic retry on rate limit."""
    for attempt in range(max_retries):
        try:
            if method == "GET":
                return client.get(endpoint, **kwargs)
            elif method == "POST":
                return client.post(endpoint, **kwargs)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limited
                wait_time = int(e.response.headers.get("Retry-After", 60))
                print(f"Rate limited. Waiting {wait_time} seconds...")
                time.sleep(wait_time)
            else:
                raise
    
    raise Exception("Max retries exceeded")
```

## Testing Authentication

Quick test script:

```python
#!/usr/bin/env python3
"""Test Storm API authentication."""

import sys
import requests
import os

def test_auth(api_key=None):
    """Test API authentication."""
    api_key = api_key or os.environ.get("STORM_API_KEY")
    
    if not api_key:
        print("❌ No API key provided")
        print("Set STORM_API_KEY environment variable or pass as argument")
        return False
    
    headers = {"storm-api-key": api_key}
    
    try:
        response = requests.get(
            "https://https://live-stargate.sionic.im/api/v2/agents",
            headers=headers,
            params={"page": 1, "size": 1}
        )
        
        if response.status_code == 200:
            print("✅ Authentication successful!")
            return True
        else:
            print(f"❌ Authentication failed: {response.status_code}")
            print(response.text)
            return False
            
    except Exception as e:
        print(f"❌ Error: {e}")
        return False

if __name__ == "__main__":
    api_key = sys.argv[1] if len(sys.argv) > 1 else None
    test_auth(api_key)
```

## Next Steps

- [Your First API Call](./03-first-api-call.ipynb) - Make your first Storm API request