# ProxyManager

ProxyManager is an asynchronous Python library for discovering, validating, and rotating HTTP(S) proxies. It wraps the entire lifecycle of working with free proxies—downloading them from a provider, filtering and validating them, persisting them to SQLite, and finally using them to make outbound requests.

## Table of Contents
- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Core Concepts](#core-concepts)
- [API Reference](#api-reference)
- [Development](#development)
- [License](#license)

## Features
- 🔁 **Automatic rotation** of proxies for any HTTP method.
- 📥 **Fetching** fresh proxies from a configurable provider endpoint.
- 🧹 **Filtering** of previously failed proxies to avoid reusing dead endpoints.
- ✅ **Validation** of proxies via `https://httpbin.org/ip` to ensure they are alive before storage.
- 💾 **Persistent storage** of active and dead proxies in an SQLite database.
- ⚙️ **Configurable timeouts and retries** to adapt to different network conditions.

## Requirements
- Python 3.7 or newer.
- `aiohttp`, `aiosqlite`, and their transitive dependencies (installed automatically when you install the package).

## Installation
Install ProxyManager directly from the repository:

```bash
# Clone the repository and install the package in editable mode
cd proxymanager
pip install -e .
```

If you prefer a regular installation without local editing, drop the `-e` flag.

## Quick Start
The example below demonstrates how to create a manager, fetch proxies, and execute an HTTP request through a rotating proxy:

```python
import asyncio
from proxymanager import ProxyManager

async def main():
    manager = ProxyManager()
    await manager.initialize_db()

    # Make a GET request via the current proxy pool.
    response = await manager.make_request_with_proxy("GET", "https://httpbin.org/ip")
    print(response)

if __name__ == "__main__":
    asyncio.run(main())
```

Run the script, and ProxyManager will automatically populate the database and reuse validated proxies for subsequent calls.

## Core Concepts
### Storage layout
- **`active_proxies`**: Contains validated proxies that can be used for requests.
- **`dead_proxies`**: Records proxies that previously failed, preventing them from being retried unless the data store is cleared.

Both tables live in an SQLite database called `proxies.db` located alongside the package modules by default. You can override the location via the `db_file` argument.

### Proxy acquisition
By default, proxies are fetched from the [ProxyScrape](https://proxyscrape.com/) public API. Pass a different `proxy_url` when constructing `ProxyManager` to integrate with a custom provider.

### Validation workflow
During a refresh cycle, proxies are:
1. Downloaded from the configured endpoint.
2. Filtered against `dead_proxies` to remove known failures.
3. Validated concurrently by issuing requests to `https://httpbin.org/ip`.
4. Stored in the `active_proxies` table for future use.

## API Reference
`ProxyManager(db_file=None, proxy_url=None, timeout=5, retries=3)`
: Configure storage, proxy source, request timeout (seconds), and fetch retries.

`await initialize_db()`
: Create the SQLite schema if it does not already exist.

`await refresh_proxies()`
: Fetch, validate, and store a fresh pool of proxies.

`await fetch_active_proxy()`
: Return a random `(ip, port)` tuple from the active proxy set, or `None` if empty.

`await make_request_with_proxy(method, url, **kwargs)`
: Issue an HTTP request using an available proxy. Additional keyword arguments are forwarded to `aiohttp.ClientSession.request` (e.g., `json`, `headers`). Dead proxies are automatically reported and retried.

`await report_dead_proxy(ip, port)`
: Manually mark a proxy as unusable; it will be moved to the `dead_proxies` table.

## Development
Clone the repository and install development dependencies:

```bash
pip install -r requirements.txt
pytest
```

Logging is enabled by default at the `INFO` level. Adjust `logging.basicConfig` in `proxymanager/manager.py` to modify verbosity during development.

## License
ProxyManager is licensed under the terms of the GNU General Public License v3.0. See [LICENSE](LICENSE) for details.
