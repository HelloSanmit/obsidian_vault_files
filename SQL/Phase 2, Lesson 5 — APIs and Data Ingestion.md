
## Phase 2, Lesson 5 — APIs and Data Ingestion

---

### Why this lesson exists in a data engineering curriculum

Every data engineering pipeline begins with data arriving from somewhere. Sometimes that somewhere is a file on disk — we covered that in Lesson 3. But in modern systems, data more often arrives through APIs — HTTP endpoints that expose data from databases, third-party services, IoT devices, payment systems, CRM platforms, and hundreds of other sources.

As a data engineer, you will spend significant time writing code that talks to APIs, extracts data from them, handles their failures gracefully, and feeds that data into your pipelines. This is called **data ingestion** — the process of pulling data from external sources into your system.

The reason this requires a dedicated lesson is that API communication introduces a category of problems that file I/O does not have. Files are local and fast. APIs are remote and unreliable. A file either exists or it does not. An API can fail in dozens of subtle ways — network timeout, rate limiting, authentication expiry, partial response, malformed JSON, server error, or simply returning different data than you expected. Writing ingestion code that handles all of these correctly is genuinely difficult.

---

### HTTP — what it is and why you need to understand it

API communication uses HTTP — HyperText Transfer Protocol. You use HTTP every time you open a browser. When you type a URL, your browser sends an HTTP request to a server. The server processes it and sends back an HTTP response. The browser renders the response.

APIs work identically — your Python code sends HTTP requests and processes responses. The only difference is that instead of a browser rendering HTML, your code parses the response data (usually JSON) and does something useful with it.

An HTTP request has several components you need to understand.

**The method** — what kind of operation you are performing:

`GET` — retrieve data. The most common in data engineering. "Give me the list of transactions." GET requests should not change any server-side data.

`POST` — send data to the server. Often used to create something or to submit a query. Some APIs use POST for complex data retrieval operations that require a request body.

`PUT` — replace an existing resource entirely.

`PATCH` — partially update an existing resource.

`DELETE` — remove a resource.

As a data engineer you will mostly use GET and occasionally POST. The others appear when you are writing data back to source systems, which is less common.

**The URL** — where you are sending the request. A URL has several components:

```
https://api.example.com/v1/students?city=Mumbai&limit=100

https://       -- protocol
api.example.com  -- host (the server)
/v1/students     -- path (which resource)
?city=Mumbai&limit=100  -- query parameters (filters and options)
```

**The headers** — metadata sent with the request. The most important for data engineering:

`Authorization` — how you prove you are allowed to access the API. Usually `Bearer <token>` for modern APIs or `Basic <base64 encoded username:password>` for older ones.

`Content-Type` — tells the server what format your request body is in. Usually `application/json` when sending JSON.

`Accept` — tells the server what format you want the response in. Usually `application/json`.

**The response** — what the server sends back. It has:

A **status code** — a number that tells you what happened. These are the ones you must know:

```
200 OK              -- success
201 Created         -- resource was created successfully
400 Bad Request     -- your request was malformed
401 Unauthorized    -- you are not authenticated
403 Forbidden       -- you are authenticated but not allowed
404 Not Found       -- the resource does not exist
429 Too Many Requests -- you hit the rate limit
500 Internal Server Error -- the server failed
502 Bad Gateway     -- the server behind the API failed
503 Service Unavailable -- the API is down
504 Gateway Timeout -- the request took too long
```

A **response body** — the actual data. Usually JSON for modern APIs.

---

### The `requests` library — Python's standard HTTP client

Python's standard library has an `http` module but it is low-level and verbose. The `requests` library provides a clean, intuitive interface and is the standard choice for HTTP communication in data engineering.

Install it:

```
pip install requests
```

The basic structure of a GET request:

```python
import requests

response = requests.get("https://api.example.com/data")

print(type(response))           # requests.models.Response
print(response.status_code)     # 200, 404, 500, etc.
print(response.headers)         # dict-like object of response headers
print(response.text)            # response body as a string
print(response.json())          # response body parsed as JSON (if it is JSON)
```

`response.json()` is a convenience method that calls `json.loads(response.text)` and returns the parsed Python object. It raises `json.JSONDecodeError` if the response body is not valid JSON.

`response.text` gives you the raw string. Use it when the response is not JSON — CSV, XML, plain text.

`response.content` gives you raw bytes. Use it for binary responses — images, files.

---

### A real API — using a public one to learn

We will use a real, public API that requires no authentication: the Open-Meteo weather API. It is free, does not need an API key, and returns well-structured JSON. Using a real API instead of a mocked one means you see real-world response structures and real-world failure modes.

```python
import requests

def get_current_weather(latitude: float, longitude: float) -> dict:
    """
    Fetch current weather data for given coordinates from the Open-Meteo API.
    
    Args:
        latitude: Geographic latitude
        longitude: Geographic longitude
    
    Returns:
        Dictionary containing weather data
    
    Raises:
        requests.HTTPError: If the API returns an error status code
        requests.ConnectionError: If connection fails
    """
    url = "https://api.open-meteo.com/v1/forecast"
    
    params = {
        "latitude": latitude,
        "longitude": longitude,
        "current": "temperature_2m,wind_speed_10m",
        "timezone": "auto",
    }
    
    response = requests.get(url, params=params, timeout=10)
    response.raise_for_status()
    return response.json()

# Mumbai coordinates
data = get_current_weather(latitude=19.0760, longitude=72.8777)
print(data)
```

Several new things here that need careful explanation.

**`params` — query parameters as a dictionary:**

Instead of manually constructing the URL string like `"https://api.open-meteo.com/v1/forecast?latitude=19.0760&longitude=72.8777&current=temperature_2m"`, you pass a dictionary to `params`. The `requests` library handles URL encoding — escaping special characters, joining with `&`, adding the `?` separator. This is safer because it handles special characters correctly and more readable because each parameter is clearly named.

**`timeout=10`:**

This is one of the most important parameters and the one most commonly forgotten. Without `timeout`, `requests.get()` will wait forever if the server never responds. In a data pipeline, a single hanging API call can block an entire pipeline indefinitely.

`timeout=10` means: if the server does not respond within 10 seconds, raise a `requests.Timeout` exception. You should always specify a timeout.

You can specify separate timeouts for connection and read:

```python
timeout=(5, 30)  # 5 seconds to connect, 30 seconds to read the response
```

Connection timeout: how long to wait to establish the TCP connection to the server. Read timeout: how long to wait for the server to send the response after connection is established. A slow API might connect quickly but take 30 seconds to process your query — you need both.

**`response.raise_for_status()`:**

This is a critical method. Without it, `requests` does not raise any exception on error status codes. If the server returns 404 or 500, `requests` happily gives you the response object and your code continues as if everything is fine.

`raise_for_status()` checks the status code. If it is 4xx or 5xx, it raises `requests.HTTPError`. If it is 2xx, it does nothing.

Always call this after every request unless you have a specific reason to handle error status codes manually.

```python
response = requests.get(url, params=params, timeout=10)
response.raise_for_status()   # raises HTTPError if status is 4xx or 5xx
data = response.json()        # only reached if status is 2xx
```

---

### Authentication — how APIs verify your identity

Most production APIs require authentication. There are three main patterns you will encounter.

**API Key — the simplest pattern:**

The API gives you a unique key string. You include it in every request, either as a query parameter or as a header.

```python
import requests

API_KEY = "your_api_key_here"

# As a query parameter -- less secure, key appears in logs
response = requests.get(
    "https://api.example.com/data",
    params={"api_key": API_KEY},
    timeout=10,
)

# As a header -- preferred, does not appear in URL logs
response = requests.get(
    "https://api.example.com/data",
    headers={"X-API-Key": API_KEY},
    timeout=10,
)
```

Never hardcode API keys in your Python files. They end up in version control and become a security breach. The correct approach is environment variables:

```python
import os
import requests

# Set in your terminal before running: set API_KEY=yourkey (Windows)
# or: export API_KEY=yourkey (Linux/Mac)
API_KEY = os.environ.get("API_KEY")

if API_KEY is None:
    raise EnvironmentError(
        "API_KEY environment variable is not set. "
        "Set it before running this script."
    )

response = requests.get(
    "https://api.example.com/data",
    headers={"X-API-Key": API_KEY},
    timeout=10,
)
```

`os.environ` is a dictionary-like object that holds environment variables. `os.environ.get("API_KEY")` returns the value if the variable exists, or `None` if it does not. We check for `None` and raise a descriptive error rather than letting the code fail mysteriously later when the key is used.

**Bearer token — the OAuth 2.0 pattern:**

Most modern APIs use OAuth 2.0. You first make a request to get a token, then include that token in subsequent requests.

```python
import requests
import os

def get_access_token(client_id: str, client_secret: str, token_url: str) -> str:
    """
    Exchange client credentials for an access token.
    This is the OAuth 2.0 client credentials flow.
    """
    response = requests.post(
        token_url,
        data={
            "grant_type": "client_credentials",
            "client_id": client_id,
            "client_secret": client_secret,
        },
        timeout=10,
    )
    response.raise_for_status()
    token_data = response.json()
    return token_data["access_token"]


def make_authenticated_request(endpoint: str, token: str) -> dict:
    """Make a request using a Bearer token for authentication."""
    response = requests.get(
        endpoint,
        headers={"Authorization": f"Bearer {token}"},
        timeout=10,
    )
    response.raise_for_status()
    return response.json()
```

The `Authorization: Bearer <token>` header is the standard way to pass a Bearer token. Every request to the API must include it.

Tokens expire — typically after an hour or a day. A robust ingestion client tracks token expiry and refreshes the token automatically when it expires. We will see this in the session management section.

**HTTP Basic Auth:**

Older APIs use Basic authentication — username and password encoded in the header.

```python
response = requests.get(
    "https://api.example.com/data",
    auth=("username", "password"),   # requests handles the encoding
    timeout=10,
)
```

`requests` accepts a tuple `(username, password)` as the `auth` parameter and handles the Base64 encoding automatically. Never do the encoding yourself.

---

### The `Session` object — why you should always use it

Every call to `requests.get()` or `requests.post()` creates a new HTTP connection from scratch. For a single request, this is fine. For ingestion code that makes thousands of requests to the same API, establishing a new TCP connection for every request wastes significant time.

The `requests.Session` object maintains a persistent connection to the server (HTTP keep-alive) and reuses it across multiple requests. It also lets you set default headers, authentication, and other configuration once rather than repeating it on every call.

```python
import requests

class APIClient:
    """
    A reusable client for a specific API.
    Uses a Session for connection reuse and consistent authentication.
    """
    
    def __init__(self, base_url: str, api_key: str, timeout: int = 10):
        self.base_url = base_url.rstrip("/")
        self.timeout = timeout
        
        # Create a session with default headers
        self._session = requests.Session()
        self._session.headers.update({
            "X-API-Key": api_key,
            "Accept": "application/json",
            "Content-Type": "application/json",
        })
    
    def get(self, path: str, params: dict = None) -> dict:
        """Make a GET request to the API."""
        url = f"{self.base_url}/{path.lstrip('/')}"
        response = self._session.get(url, params=params, timeout=self.timeout)
        response.raise_for_status()
        return response.json()
    
    def post(self, path: str, payload: dict) -> dict:
        """Make a POST request to the API."""
        import json
        url = f"{self.base_url}/{path.lstrip('/')}"
        response = self._session.post(
            url,
            data=json.dumps(payload),
            timeout=self.timeout
        )
        response.raise_for_status()
        return response.json()
    
    def close(self) -> None:
        """Close the session and release the connection."""
        self._session.close()
    
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.close()
        return False


# Using the client as a context manager
with APIClient("https://api.example.com", api_key="key123") as client:
    students = client.get("/students", params={"city": "Mumbai"})
    print(students)
```

`self.base_url.rstrip("/")` removes trailing slashes from the base URL. `path.lstrip("/")` removes leading slashes from the path. Combined, `f"{self.base_url}/{path.lstrip('/')}"` always produces a correctly joined URL regardless of whether the caller included slashes.

The `__enter__` and `__exit__` methods make the client a context manager — when the `with` block exits, `close()` is called and the session is properly cleaned up. Connection objects that are not closed are resource leaks.

---

### Pagination — the most important API concept for data engineers

This is the concept that separates data engineering API work from ordinary web development, and it is the one most commonly handled incorrectly.

APIs almost never return all available data in a single response. If an API has 1 million customer records, returning them all in one response would be slow and memory-intensive for both the server and the client. Instead, APIs use **pagination** — returning a fixed number of records per response and providing a way to request the next page.

There are three common pagination patterns.

**Offset-based pagination:**

The API accepts `page` and `limit` (or `offset` and `limit`) parameters. You request page 1, get records 1-100. You request page 2, get records 101-200. And so on.

```python
import requests
import time

def fetch_all_records_offset(base_url: str, api_key: str, page_size: int = 100) -> list[dict]:
    """
    Fetch all records from an offset-paginated API.
    
    The API uses ?page=N&limit=N parameters.
    The API signals "no more pages" by returning an empty list.
    """
    all_records = []
    page = 1
    
    with requests.Session() as session:
        session.headers["X-API-Key"] = api_key
        
        while True:
            print(f"Fetching page {page}...")
            
            response = session.get(
                f"{base_url}/records",
                params={"page": page, "limit": page_size},
                timeout=30,
            )
            response.raise_for_status()
            data = response.json()
            
            # Different APIs signal end differently:
            # Some return an empty list
            records = data.get("records", data if isinstance(data, list) else [])
            
            if not records:
                print(f"No more records. Stopping at page {page}.")
                break
            
            all_records.extend(records)
            print(f"  Got {len(records)} records. Total so far: {len(all_records)}")
            
            # Check if this was the last page
            # Some APIs tell you the total: data.get("total_count")
            if len(records) < page_size:
                print("Partial page received -- this is the last page.")
                break
            
            page += 1
            time.sleep(0.1)   # be polite to the server
    
    return all_records
```

The `while True` loop with explicit break conditions is the correct pattern for pagination. You do not know in advance how many pages there are, so you cannot use a `for range()` loop. You loop until the API signals there is nothing more — either an empty response, a `has_more: false` field, or a page smaller than the requested size.

`time.sleep(0.1)` between requests is called **rate limiting** on the client side. We will cover this properly in a moment, but the principle is: APIs have limits on how many requests per second they accept. Sending 1000 requests per second will get your API key blocked. Being polite costs almost nothing and avoids a lot of pain.

**Cursor-based pagination:**

Instead of page numbers, the API returns a cursor — a token that represents your current position. You pass the cursor in your next request to get the next batch.

```python
def fetch_all_records_cursor(base_url: str, api_key: str, page_size: int = 100) -> list[dict]:
    """
    Fetch all records from a cursor-paginated API.
    
    The API returns a 'next_cursor' field. When it is None/absent, there are no more pages.
    Cursor pagination is more reliable than offset pagination because it handles
    concurrent inserts correctly -- new records added while you paginate do not
    cause you to skip records or see duplicates.
    """
    all_records = []
    cursor = None
    
    with requests.Session() as session:
        session.headers["X-API-Key"] = api_key
        
        while True:
            params = {"limit": page_size}
            if cursor:
                params["cursor"] = cursor
            
            response = session.get(
                f"{base_url}/records",
                params=params,
                timeout=30,
            )
            response.raise_for_status()
            data = response.json()
            
            records = data.get("records", [])
            all_records.extend(records)
            
            cursor = data.get("next_cursor")
            
            print(f"Got {len(records)} records. Cursor: {cursor}. Total: {len(all_records)}")
            
            if not cursor or not records:
                break
    
    return all_records
```

Why cursor pagination is superior to offset pagination from a data engineering perspective:

With offset pagination, if a new record is inserted between your page 1 and page 2 requests, every subsequent record shifts by one position. Record at offset 100 is now at offset 101. Your page 2 request starts at offset 100 and skips the record that was at offset 100 before the insert. You silently miss data.

With cursor pagination, the cursor points to a specific position in the underlying data structure (often a database index). New inserts do not affect the cursor's meaning. You always get the next batch from exactly where you left off.

In data engineering, you are often ingesting from live systems where new records are constantly being added. Cursor pagination guarantees completeness. Offset pagination does not.

**Link header pagination:**

Some APIs follow the RFC 5988 standard and include pagination links in the response headers rather than the body.

```python
def fetch_all_records_link_header(initial_url: str, api_key: str) -> list[dict]:
    """
    Fetch all records from an API that uses Link headers for pagination.
    The 'next' link in the Link header contains the URL for the next page.
    """
    all_records = []
    url = initial_url
    
    with requests.Session() as session:
        session.headers["Authorization"] = f"Bearer {api_key}"
        
        while url:
            response = session.get(url, timeout=30)
            response.raise_for_status()
            
            records = response.json()
            all_records.extend(records if isinstance(records, list) else [records])
            
            # Parse the Link header to find the next URL
            url = parse_next_link(response.headers.get("Link", ""))
            print(f"Got {len(all_records)} total records. Next URL: {url}")
    
    return all_records


def parse_next_link(link_header: str) -> str | None:
    """
    Parse a Link header and return the 'next' URL if present.
    
    Link header format:
    <https://api.example.com/page2>; rel="next", <https://api.example.com/page1>; rel="prev"
    """
    if not link_header:
        return None
    
    for part in link_header.split(","):
        part = part.strip()
        if 'rel="next"' in part:
            url_part = part.split(";")[0].strip()
            if url_part.startswith("<") and url_part.endswith(">"):
                return url_part[1:-1]   # remove the < and >
    
    return None
```

String slicing `url_part[1:-1]` removes the first character (`<`) and last character (`>`) from the URL string. `[1:-1]` starts at index 1 (skipping index 0 which is `<`) and ends at index -1 (excluding the last character which is `>`). This is Python's standard slicing syntax: `string[start:end]` where negative indices count from the end.

---

### Rate limiting — one of the most important concepts for data engineers

APIs limit how many requests you can make within a time window. Common limits:

- 1000 requests per hour
- 100 requests per minute
- 10 requests per second

When you exceed the limit, the API returns a `429 Too Many Requests` response. Some APIs include a `Retry-After` header telling you how many seconds to wait before trying again.

Naive code that ignores rate limits will get your API key blocked. Even if it does not, hammering an API with thousands of rapid requests is impolite and strains the server.

There are two levels of rate limit handling.

**Proactive** — respect the limit before hitting it by adding delays between requests.

**Reactive** — detect when you have hit the limit and back off appropriately.

```python
import requests
import time
import random
from requests.exceptions import HTTPError


def make_request_with_retry(
    session: requests.Session,
    url: str,
    params: dict = None,
    max_retries: int = 5,
    base_delay: float = 1.0,
) -> dict:
    """
    Make an HTTP GET request with automatic retry on transient failures.
    
    Uses exponential backoff with jitter -- the industry standard pattern
    for retrying API requests.
    
    Args:
        session: requests.Session to use for the request
        url: The endpoint URL
        params: Optional query parameters
        max_retries: Maximum number of retry attempts
        base_delay: Initial delay in seconds (doubles each retry)
    
    Returns:
        Parsed JSON response as a dictionary
    
    Raises:
        HTTPError: If a non-retryable error occurs (4xx except 429)
        Exception: If all retry attempts are exhausted
    """
    last_exception = None
    
    for attempt in range(1, max_retries + 1):
        try:
            response = session.get(url, params=params, timeout=30)
            
            if response.status_code == 429:
                # Rate limited -- check for Retry-After header
                retry_after = response.headers.get("Retry-After")
                
                if retry_after:
                    wait_seconds = float(retry_after)
                    print(f"Rate limited. Server says wait {wait_seconds}s.")
                else:
                    # No Retry-After header -- use exponential backoff
                    wait_seconds = base_delay * (2 ** (attempt - 1))
                    print(f"Rate limited. Backing off for {wait_seconds:.1f}s.")
                
                time.sleep(wait_seconds)
                continue
            
            if response.status_code in (500, 502, 503, 504):
                # Server error -- these are transient and worth retrying
                wait_seconds = base_delay * (2 ** (attempt - 1))
                jitter = random.uniform(0, wait_seconds * 0.1)
                total_wait = wait_seconds + jitter
                print(
                    f"Attempt {attempt}/{max_retries}: "
                    f"Server error {response.status_code}. "
                    f"Retrying in {total_wait:.1f}s."
                )
                time.sleep(total_wait)
                continue
            
            # For 4xx errors (except 429), do not retry -- it will not help
            response.raise_for_status()
            return response.json()
        
        except requests.ConnectionError as e:
            # Network-level failure -- worth retrying
            wait_seconds = base_delay * (2 ** (attempt - 1))
            print(f"Attempt {attempt}/{max_retries}: Connection failed. Retrying in {wait_seconds:.1f}s.")
            last_exception = e
            time.sleep(wait_seconds)
        
        except requests.Timeout as e:
            # Request timed out -- worth retrying
            wait_seconds = base_delay * (2 ** (attempt - 1))
            print(f"Attempt {attempt}/{max_retries}: Timeout. Retrying in {wait_seconds:.1f}s.")
            last_exception = e
            time.sleep(wait_seconds)
    
    raise Exception(
        f"All {max_retries} retry attempts exhausted. Last error: {last_exception}"
    )
```

**Exponential backoff** means each retry waits twice as long as the previous one. If the base delay is 1 second:

- Attempt 1 fails → wait 1 second
- Attempt 2 fails → wait 2 seconds
- Attempt 3 fails → wait 4 seconds
- Attempt 4 fails → wait 8 seconds
- Attempt 5 fails → wait 16 seconds

Why exponential? If a server is overloaded, dozens of clients retrying immediately will make it worse. Spreading retries out over time gives the server space to recover.

**Jitter** — the `random.uniform(0, wait_seconds * 0.1)` addition — adds a small random amount to each wait time. Without jitter, if many clients are all retrying with the same exponential backoff, they will all retry at exactly the same moment (the "thundering herd" problem), immediately overloading the server again. Jitter spreads the retries across time so they do not all hit simultaneously.

`random.uniform(a, b)` returns a random float uniformly distributed between `a` and `b`. It is from Python's `random` module.

**Which errors to retry and which not to:**

`429 Too Many Requests` — retry after waiting. The request was valid, the server just needs you to slow down.

`500, 502, 503, 504` — retry. These are server-side failures that are typically transient — the server crashed, was restarted, or was temporarily unavailable.

`400 Bad Request` — do not retry. Your request was malformed. Retrying the same malformed request will always get the same error. Fix the request instead.

`401 Unauthorized` — do not retry (unless you are refreshing a token). Your credentials are wrong.

`403 Forbidden` — do not retry. You are not allowed to access this resource. Retrying will not change that.

`404 Not Found` — do not retry. The resource does not exist. Retrying will not create it.

---

### Building a complete ingestion client

This brings everything together into a production-quality ingestion client — the kind of thing you would actually write for a real data pipeline.

```python
import requests
import time
import json
import os
import random
from pathlib import Path
from datetime import datetime


class WeatherIngestionClient:
    """
    Ingestion client for the Open-Meteo weather API.
    
    Demonstrates a complete, production-quality API client with:
    - Session management
    - Automatic retry with exponential backoff and jitter
    - Structured logging of each request
    - Output to Parquet via Pandas
    """

    BASE_URL = "https://api.open-meteo.com/v1"
    
    CITY_COORDINATES = {
        "Mumbai":    (19.0760, 72.8777),
        "Delhi":     (28.6139, 77.2090),
        "Chennai":   (13.0827, 80.2707),
        "Bangalore": (12.9716, 77.5946),
        "Hyderabad": (17.3850, 78.4867),
    }

    def __init__(self, timeout: int = 15, max_retries: int = 3):
        self.timeout = timeout
        self.max_retries = max_retries
        self._session = requests.Session()
        self._session.headers.update({"Accept": "application/json"})
        self._request_log: list[dict] = []

    def _make_request(self, path: str, params: dict) -> dict:
        """
        Make a GET request with retry logic.
        Logs every attempt for observability.
        """
        url = f"{self.BASE_URL}/{path.lstrip('/')}"
        
        for attempt in range(1, self.max_retries + 1):
            request_start = datetime.now()
            
            try:
                response = self._session.get(url, params=params, timeout=self.timeout)
                
                self._request_log.append({
                    "timestamp": request_start.isoformat(),
                    "url": url,
                    "status_code": response.status_code,
                    "attempt": attempt,
                    "duration_ms": round(
                        (datetime.now() - request_start).total_seconds() * 1000, 2
                    ),
                })
                
                if response.status_code == 429:
                    wait = float(response.headers.get("Retry-After", 2 ** attempt))
                    print(f"  Rate limited. Waiting {wait:.1f}s...")
                    time.sleep(wait)
                    continue
                
                if response.status_code >= 500:
                    wait = (2 ** (attempt - 1)) + random.uniform(0, 0.5)
                    print(f"  Server error {response.status_code}. Retry {attempt}/{self.max_retries} in {wait:.1f}s...")
                    time.sleep(wait)
                    continue
                
                response.raise_for_status()
                return response.json()
            
            except (requests.ConnectionError, requests.Timeout) as e:
                wait = (2 ** (attempt - 1)) + random.uniform(0, 0.5)
                print(f"  Network error: {e}. Retry {attempt}/{self.max_retries} in {wait:.1f}s...")
                self._request_log.append({
                    "timestamp": request_start.isoformat(),
                    "url": url,
                    "status_code": None,
                    "attempt": attempt,
                    "error": str(e),
                })
                if attempt < self.max_retries:
                    time.sleep(wait)
        
        raise RuntimeError(f"All {self.max_retries} attempts failed for {url}")

    def fetch_weather(self, city: str) -> dict | None:
        """
        Fetch current weather for a named city.
        Returns None if the city is not in our coordinate registry.
        """
        if city not in self.CITY_COORDINATES:
            print(f"Unknown city: {city}. Available: {list(self.CITY_COORDINATES)}")
            return None
        
        lat, lon = self.CITY_COORDINATES[city]
        
        print(f"Fetching weather for {city}...")
        raw = self._make_request(
            path="forecast",
            params={
                "latitude": lat,
                "longitude": lon,
                "current": "temperature_2m,wind_speed_10m,precipitation",
                "timezone": "auto",
            },
        )
        
        current = raw.get("current", {})
        return {
            "city": city,
            "latitude": lat,
            "longitude": lon,
            "fetched_at": datetime.now().isoformat(),
            "temperature_c": current.get("temperature_2m"),
            "wind_speed_kmh": current.get("wind_speed_10m"),
            "precipitation_mm": current.get("precipitation"),
        }

    def fetch_all_cities(self, delay_between: float = 0.5) -> list[dict]:
        """
        Fetch weather for all cities in the coordinate registry.
        Adds a delay between requests to be polite to the API.
        """
        results = []
        cities = list(self.CITY_COORDINATES.keys())
        
        for i, city in enumerate(cities):
            record = self.fetch_weather(city)
            if record:
                results.append(record)
                print(f"  Temperature: {record['temperature_c']}°C, "
                      f"Wind: {record['wind_speed_kmh']} km/h")
            
            if i < len(cities) - 1:
                time.sleep(delay_between)
        
        return results

    def save_to_parquet(self, records: list[dict], output_path: str) -> None:
        """Save fetched records to a Parquet file."""
        import pandas as pd
        Path(output_path).parent.mkdir(parents=True, exist_ok=True)
        df = pd.DataFrame(records)
        df.to_parquet(output_path, compression="snappy", index=False)
        print(f"Saved {len(records)} records to {output_path}")

    def save_request_log(self, output_path: str) -> None:
        """Save the request log as JSON for observability."""
        Path(output_path).parent.mkdir(parents=True, exist_ok=True)
        with open(output_path, "w", encoding="utf-8") as f:
            json.dump(self._request_log, f, indent=2)
        print(f"Request log saved: {len(self._request_log)} requests logged")

    def close(self) -> None:
        self._session.close()

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.close()
        return False


# Run the ingestion
with WeatherIngestionClient(timeout=15, max_retries=3) as client:
    print("Starting weather data ingestion...")
    weather_records = client.fetch_all_cities(delay_between=0.5)
    
    if weather_records:
        client.save_to_parquet(weather_records, "data/weather/current.parquet")
        client.save_request_log("data/weather/request_log.json")
        
        print(f"\nIngestion complete. Fetched {len(weather_records)} city records.")
        
        import pandas as pd
        df = pd.read_parquet("data/weather/current.parquet")
        print("\nFetched data:")
        print(df.to_string(index=False))
```

Run this. It makes real HTTP requests to a real API. Look at the data it returns. Look at the request log JSON it creates.

Several things to examine in this implementation:

`CITY_COORDINATES` is a class attribute — a dictionary shared across all instances. It is constant configuration that belongs at the class level, not repeated in every instance.

`_request_log` is an instance attribute — each client instance maintains its own log of requests. This is the right level because different client instances might be talking to different APIs.

The `_make_request` method is the single point of contact with the HTTP layer. Every request goes through it. This means retry logic, logging, and error handling are implemented once and applied everywhere. This is the Single Responsibility Principle — the method's one job is to make a reliable HTTP request.

The `fetch_weather` method has no HTTP knowledge — it calls `_make_request` and interprets the result. Separation of concerns.

---

### Inspecting responses — what to do when an API returns unexpected data

Real APIs return unexpected data. Fields are missing, types are wrong, values are null when they should not be. Writing ingestion code that crashes on the first unexpected response is not production quality.

```python
def safely_extract(data: dict, path: list[str], default=None):
    """
    Safely extract a nested value from a dictionary using a path of keys.
    Returns default if any key in the path is missing.
    
    Example:
        data = {"current": {"temperature_2m": 28.5}}
        safely_extract(data, ["current", "temperature_2m"])  # 28.5
        safely_extract(data, ["current", "humidity"])        # None
        safely_extract(data, ["forecast", "daily"])          # None
    """
    current = data
    for key in path:
        if not isinstance(current, dict):
            return default
        current = current.get(key, default)
        if current is default:
            return default
    return current


# Testing with weather API response structure
response_data = {
    "latitude": 19.076,
    "longitude": 72.877,
    "current": {
        "temperature_2m": 28.5,
        "wind_speed_10m": 12.3,
    }
}

print(safely_extract(response_data, ["current", "temperature_2m"]))   # 28.5
print(safely_extract(response_data, ["current", "humidity"]))          # None
print(safely_extract(response_data, ["forecast", "daily", "rain"]))    # None
```

`isinstance(current, dict)` checks whether the current value is a dictionary before trying to call `.get()` on it. If `current` is a string or a list, calling `.get()` would raise an `AttributeError`. The check prevents this.

---

### New built-in functions and methods introduced in this lesson

You asked me to explain any new built-ins or methods. Here are the ones that appeared in this lesson that were not in previous lessons.

**`os.environ`** — a dictionary-like object (`os._Environ`) mapping environment variable names to their values. `os.environ.get("KEY")` returns the value or `None`. `os.environ["KEY"]` returns the value or raises `KeyError`. Use `.get()` when the variable might not exist, use `[]` when it must exist.

**`time.sleep(seconds)`** — pauses execution for the specified number of seconds. `seconds` can be a float — `time.sleep(0.5)` pauses for 500 milliseconds. During the pause, the Python thread is yielded to the OS, consuming no CPU.

**`random.uniform(a, b)`** — returns a random float `N` such that `a <= N <= b`. Used for jitter in retry logic. From the `random` module.

**`string.rstrip(chars)`** and **`string.lstrip(chars)`** — remove characters from the right or left end of a string. `"https://api.com/".rstrip("/")` returns `"https://api.com"`. The argument is a set of characters to remove, not a substring — `"aabba".rstrip("a")` removes trailing `a` characters, giving `"aabb"`.

**`str.split(separator, maxsplit)`** — splits a string on the separator. Without arguments, splits on any whitespace and removes empty strings. `"hello world".split()` gives `["hello", "world"]`. `"a,b,c".split(",")` gives `["a", "b", "c"]`. `"a,b,c".split(",", 1)` gives `["a", "b,c"]` — splits at most once.

**`response.headers`** — a case-insensitive dictionary of HTTP response headers. `response.headers["Content-Type"]` and `response.headers["content-type"]` return the same value.

**`response.raise_for_status()`** — raises `requests.HTTPError` if the response status code is 4xx or 5xx. Does nothing for 2xx or 3xx. The single most important method for correct error handling.

**`response.json()`** — parses the response body as JSON. Equivalent to `json.loads(response.text)`. Raises `json.JSONDecodeError` if the response is not valid JSON. Raises `requests.exceptions.JSONDecodeError` in newer versions of requests (which inherits from `json.JSONDecodeError`).

**`datetime.now()`** — returns the current local date and time as a `datetime` object. `datetime.now().isoformat()` converts it to the ISO 8601 string format: `"2024-01-15T14:30:00.123456"`. From the `datetime` module's `datetime` class.

**`(datetime.now() - earlier_datetime).total_seconds()`** — `datetime` objects support subtraction. Subtracting two `datetime` objects gives a `timedelta` object. `.total_seconds()` on a `timedelta` gives the total difference in seconds as a float, including fractional seconds.

---

### Exercises — Lesson 5

1. Write a function `fetch_weather_for_cities` that accepts a list of city dictionaries, each with `name`, `latitude`, and `longitude` keys. It should fetch the current temperature and wind speed for each city from the Open-Meteo API (use the same URL structure as in the lesson). It must handle individual city failures gracefully — if one city fails, continue with the others and record the error. Return a dataclass `IngestionResult` with `successful_records` (list of dicts), `failed_cities` (list of city names), and `errors` (list of error messages).
    
2. Write a function `paginate_mock_api` that simulates fetching from a paginated offset-based API. Create a function `mock_api_server` that takes a page number and page size and returns a fake response dictionary with a `records` key and a `has_more` key. The `mock_api_server` should return 3 full pages of 5 records each and then return an empty final page. Write `paginate_mock_api` to call `mock_api_server` repeatedly until `has_more` is False, collecting all records. This teaches the pagination logic without needing a real API.
    
3. Implement a class `RateLimitedSession` that wraps `requests.Session` and enforces a maximum number of requests per second. It should track when the last request was made and automatically `time.sleep()` if the next request would exceed the rate limit. Add a method `get(url, **kwargs)` that respects the rate limit before making the actual request. Test it by making 10 requests in a loop and verifying the total time is consistent with the rate limit.
    
4. Write a function `validate_api_response` that accepts a response dictionary and a schema dictionary. The schema maps field names to their expected Python types (e.g., `{"temperature": float, "city": str}`). The function should return a `ValidationResult` dataclass with `is_valid`, `missing_fields`, `type_mismatches`, and `extra_fields`. Test it with both valid and invalid response structures.
    

---

### Answers — Lesson 5

**Exercise 1**

```python
import requests
import time
from dataclasses import dataclass, field


@dataclass
class IngestionResult:
    successful_records: list[dict] = field(default_factory=list)
    failed_cities: list[str] = field(default_factory=list)
    errors: list[str] = field(default_factory=list)

    @property
    def success_rate(self) -> float:
        total = len(self.successful_records) + len(self.failed_cities)
        if total == 0:
            return 0.0
        return len(self.successful_records) / total * 100


def fetch_weather_for_cities(
    cities: list[dict],
    delay_between: float = 0.5,
) -> IngestionResult:
    """
    Fetch current weather for a list of cities from Open-Meteo.
    
    Args:
        cities: List of dicts with 'name', 'latitude', 'longitude' keys
        delay_between: Seconds to wait between requests
    
    Returns:
        IngestionResult with successful records and any errors
    """
    result = IngestionResult()
    
    with requests.Session() as session:
        session.headers["Accept"] = "application/json"
        
        for i, city in enumerate(cities):
            city_name = city.get("name", "Unknown")
            
            try:
                response = session.get(
                    "https://api.open-meteo.com/v1/forecast",
                    params={
                        "latitude": city["latitude"],
                        "longitude": city["longitude"],
                        "current": "temperature_2m,wind_speed_10m",
                        "timezone": "auto",
                    },
                    timeout=15,
                )
                response.raise_for_status()
                data = response.json()
                
                current = data.get("current", {})
                record = {
                    "city": city_name,
                    "latitude": city["latitude"],
                    "longitude": city["longitude"],
                    "temperature_c": current.get("temperature_2m"),
                    "wind_speed_kmh": current.get("wind_speed_10m"),
                }
                result.successful_records.append(record)
                print(f"[OK] {city_name}: {record['temperature_c']}°C")
            
            except requests.HTTPError as e:
                msg = f"{city_name}: HTTP {e.response.status_code} — {e}"
                result.failed_cities.append(city_name)
                result.errors.append(msg)
                print(f"[FAIL] {msg}")
            
            except (requests.ConnectionError, requests.Timeout) as e:
                msg = f"{city_name}: Network error — {e}"
                result.failed_cities.append(city_name)
                result.errors.append(msg)
                print(f"[FAIL] {msg}")
            
            except Exception as e:
                msg = f"{city_name}: Unexpected error — {e}"
                result.failed_cities.append(city_name)
                result.errors.append(msg)
                print(f"[FAIL] {msg}")
            
            if i < len(cities) - 1:
                time.sleep(delay_between)
    
    return result


cities = [
    {"name": "Mumbai",   "latitude": 19.0760, "longitude": 72.8777},
    {"name": "Delhi",    "latitude": 28.6139, "longitude": 77.2090},
    {"name": "Chennai",  "latitude": 13.0827, "longitude": 80.2707},
]

result = fetch_weather_for_cities(cities)
print(f"\nSuccess: {len(result.successful_records)} cities")
print(f"Failed: {len(result.failed_cities)} cities")
print(f"Success rate: {result.success_rate:.1f}%")
for record in result.successful_records:
    print(f"  {record}")
```

**Exercise 2**

```python
def mock_api_server(page: int, page_size: int) -> dict:
    """
    Simulates a paginated API server response.
    Returns 3 full pages then an empty final page.
    """
    total_records = 15
    start_index = (page - 1) * page_size
    
    if start_index >= total_records:
        return {"records": [], "has_more": False, "page": page}
    
    end_index = min(start_index + page_size, total_records)
    records = [
        {
            "id": i + 1,
            "student_id": 1000 + i,
            "name": f"Student {i + 1}",
            "gpa": round(6.0 + (i % 4) * 0.8, 1),
        }
        for i in range(start_index, end_index)
    ]
    
    has_more = end_index < total_records
    return {"records": records, "has_more": has_more, "page": page}


def paginate_mock_api(page_size: int = 5) -> list[dict]:
    """
    Fetch all records from the mock paginated API.
    Continues fetching until has_more is False.
    """
    all_records = []
    page = 1
    
    while True:
        print(f"Fetching page {page}...")
        response = mock_api_server(page=page, page_size=page_size)
        
        records = response.get("records", [])
        has_more = response.get("has_more", False)
        
        all_records.extend(records)
        print(f"  Got {len(records)} records. Has more: {has_more}. Total: {len(all_records)}")
        
        if not has_more:
            print("No more pages.")
            break
        
        page += 1
    
    return all_records


all_records = paginate_mock_api(page_size=5)
print(f"\nTotal records fetched: {len(all_records)}")
for record in all_records:
    print(f"  {record}")
```

**Exercise 3**

```python
import requests
import time


class RateLimitedSession:
    """
    A requests.Session wrapper that enforces a maximum requests-per-second rate.
    Automatically sleeps if the rate limit would be exceeded.
    """
    
    def __init__(self, max_requests_per_second: float = 2.0):
        self._session = requests.Session()
        self._min_interval = 1.0 / max_requests_per_second
        self._last_request_time: float = 0.0
        self._request_count = 0
    
    def _wait_if_needed(self) -> None:
        """Sleep if necessary to respect the rate limit."""
        now = time.time()
        elapsed = now - self._last_request_time
        
        if elapsed < self._min_interval:
            sleep_for = self._min_interval - elapsed
            print(f"  Rate limit: sleeping {sleep_for:.3f}s")
            time.sleep(sleep_for)
        
        self._last_request_time = time.time()
    
    def get(self, url: str, **kwargs) -> requests.Response:
        """Make a rate-limited GET request."""
        self._wait_if_needed()
        self._request_count += 1
        print(f"  Request #{self._request_count}: GET {url[:60]}...")
        return self._session.get(url, **kwargs)
    
    def close(self) -> None:
        self._session.close()
    
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.close()
        return False


# Test: 5 requests at 2 requests/second should take ~2 seconds minimum
with RateLimitedSession(max_requests_per_second=2.0) as session:
    start = time.time()
    
    for i in range(5):
        response = session.get(
            "https://api.open-meteo.com/v1/forecast",
            params={"latitude": 19.076, "longitude": 72.877, "current": "temperature_2m"},
            timeout=15,
        )
        print(f"  Status: {response.status_code}")
    
    total_time = time.time() - start
    print(f"\nTotal time for 5 requests at 2/sec: {total_time:.2f}s")
    print(f"Expected minimum: {4 * 0.5:.1f}s (4 intervals of 0.5s)")
```

**Exercise 4**

```python
from dataclasses import dataclass, field


@dataclass
class ValidationResult:
    is_valid: bool = True
    missing_fields: list[str] = field(default_factory=list)
    type_mismatches: list[str] = field(default_factory=list)
    extra_fields: list[str] = field(default_factory=list)

    def __post_init__(self):
        self.is_valid = (
            not self.missing_fields and not self.type_mismatches
        )

    def __str__(self) -> str:
        if self.is_valid:
            extras = f" ({len(self.extra_fields)} extra fields)" if self.extra_fields else ""
            return f"Valid{extras}"
        issues = self.missing_fields + self.type_mismatches
        return f"Invalid: {'; '.join(issues)}"


def validate_api_response(response: dict, schema: dict[str, type]) -> ValidationResult:
    """
    Validate an API response dictionary against an expected schema.
    
    Args:
        response: The API response dictionary to validate
        schema: Maps field names to their expected Python types
    
    Returns:
        ValidationResult with details of any issues found
    """
    missing = []
    mismatches = []
    
    for field_name, expected_type in schema.items():
        if field_name not in response:
            missing.append(field_name)
        else:
            value = response[field_name]
            if value is not None and not isinstance(value, expected_type):
                mismatches.append(
                    f"'{field_name}': expected {expected_type.__name__}, "
                    f"got {type(value).__name__} ({value!r})"
                )
    
    extra = [k for k in response if k not in schema]
    
    return ValidationResult(
        missing_fields=missing,
        type_mismatches=mismatches,
        extra_fields=extra,
    )


weather_schema = {
    "city": str,
    "temperature_c": float,
    "wind_speed_kmh": float,
    "latitude": float,
    "longitude": float,
}

valid_response = {
    "city": "Mumbai",
    "temperature_c": 28.5,
    "wind_speed_kmh": 12.3,
    "latitude": 19.076,
    "longitude": 72.877,
}

invalid_response = {
    "city": "Mumbai",
    "temperature_c": "28.5",   # string instead of float
    "latitude": 19.076,
    "extra_field": "unexpected",
    # missing wind_speed_kmh and longitude
}

print("Valid response:")
result = validate_api_response(valid_response, weather_schema)
print(f"  {result}")

print("\nInvalid response:")
result = validate_api_response(invalid_response, weather_schema)
print(f"  {result}")
print(f"  Missing: {result.missing_fields}")
print(f"  Type mismatches: {result.type_mismatches}")
print(f"  Extra fields: {result.extra_fields}")
```

Lesson 5 is complete. Tell me when you are ready for Lesson 6.