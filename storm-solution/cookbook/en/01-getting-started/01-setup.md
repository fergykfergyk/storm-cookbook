# Installation & Setup

Get started with Storm API in minutes.

## Prerequisites

- Python 3.7 or higher
- pip package manager
- Storm API key

## Installation

### Install Required Libraries

```bash
pip install requests
```

For advanced features:

```bash
pip install requests pandas numpy jupyter
```

## API Configuration

### Base URL

Storm API is available at:

```
https://https://live-stargate.sionic.im
```

### Authentication

Storm API uses header-based authentication:

```python
headers = {
    "storm-api-key": "your-api-key-here"
}
```

## Environment Setup

### Using Environment Variables

Create a `.env` file:

```bash
# .env
STORM_API_KEY=your-api-key-here
STORM_API_URL=https://https://live-stargate.sionic.im
```

Load in Python:

```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv("STORM_API_KEY")
API_URL = os.getenv("STORM_API_URL", "https://https://live-stargate.sionic.im")
```

### Configuration Class

```python
class StormConfig:
    def __init__(self, api_key=None, api_url=None):
        self.api_key = api_key or os.getenv("STORM_API_KEY")
        self.api_url = api_url or os.getenv("STORM_API_URL", "https://https://live-stargate.sionic.im")
        
        if not self.api_key:
            raise ValueError("Storm API key is required")
    
    @property
    def headers(self):
        return {"storm-api-key": self.api_key}

# Usage
config = StormConfig()
```

## Verify Installation

Test your setup:

```python
import requests

def test_connection(config):
    """Test Storm API connection."""
    try:
        # Test with agent list endpoint
        response = requests.get(
            f"{config.api_url}/api/v2/agents",
            headers=config.headers
        )
        
        if response.status_code == 200:
            print("✅ Successfully connected to Storm API")
            data = response.json()
            print(f"📊 Found {len(data['data']['data'])} agents")
            return True
        else:
            print(f"❌ Connection failed: {response.status_code}")
            return False
    except Exception as e:
        print(f"❌ Error: {e}")
        return False

# Test
config = StormConfig()
test_connection(config)
```

## Next Steps

- [Authentication Guide](./02-authentication.md)
- [Your First API Call](./03-first-api-call.ipynb)