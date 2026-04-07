# CLAUDE.md

## Project Overview

Python library for interacting with Anker Solix power devices (Solarbank, Inverter, PPS, etc.) via reverse-engineered cloud and MQTT APIs. Async-first, built on `aiohttp`. Not a packaged library (`package-mode = false`).

- **Version**: 3.4.0
- **Python**: >= 3.12
- **License**: MIT
- **Package Manager**: Poetry >= 2.1

## Repository Structure

```
api/                  # Core library (24 modules, ~30k lines)
  api.py              # Main AnkerSolixApi class (entry point)
  apibase.py          # Base class with cache management
  session.py          # HTTP client session & authentication
  apitypes.py         # Types, enums, dataclasses, API endpoints
  errors.py           # Custom exception hierarchy
  energy.py           # Energy analysis & statistics
  schedule.py         # Device parameter scheduling
  vehicle.py          # Vehicle management
  export.py           # Data export to JSON
  hesapi.py           # HES charging service endpoints
  powerpanel.py       # Power panel endpoints
  poller.py           # Data poller for cache updates
  helpers.py          # Utility functions
  mqtt.py             # MQTT session management
  mqtt_device.py      # Base MQTT device control
  mqtt_solarbank.py   # Solarbank MQTT controls
  mqtt_charger.py     # Charger MQTT controls
  mqtt_pps.py         # Portable Power Station MQTT controls
  mqtt_various.py     # Misc device MQTT controls
  mqtt_factory.py     # MQTT device factory
  mqttmap.py          # MQTT field conversion mappings
  mqttcmdmap.py       # MQTT command mappings
  mqtttypes.py        # MQTT type definitions
examples/             # 22+ example JSON data dirs for offline testing
docs/                 # Additional documentation
*.py (root)           # Executable scripts (monitor, export, test, etc.)
common.py             # Shared utilities for root scripts (logging, credentials)
```

## Key Architecture

- **Async/await throughout** - all API calls use `aiohttp`/`asyncio`
- **Class hierarchy**: `AnkerSolixBaseApi` (apibase.py) -> `AnkerSolixApi` (api.py)
- **Feature modules** (energy, schedule, vehicle, export) are mixed into the main API class
- **MQTT layer** has its own type system (`mqtttypes.py`) and device-specific control classes
- **Session handling** via `AnkerSolixClientSession` wrapping aiohttp with auth/encryption

## Dependencies

```
aiohttp (>=3.10.11)       # Async HTTP
aiofiles (>=23.2.0)       # Async file I/O
cryptography (>=3.4.8)    # Encryption
paho-mqtt (>=2.1.0)       # MQTT protocol
python-dotenv (>=1.2.1)   # .env file support
```

## Development Setup

```bash
# Install with Poetry
poetry install

# Or manually
pip install -r requirements.txt  # if available
pip install aiohttp aiofiles cryptography paho-mqtt python-dotenv
```

## Code Quality Tools

All configured via pre-commit hooks (`.pre-commit-config.yaml`):

| Tool       | Version       | Purpose                |
|------------|---------------|------------------------|
| **isort**  | 5.13.2        | Import sorting (black profile) |
| **black**  | 24.2.0        | Code formatting        |
| **flake8** | 7.0.0         | Linting                |
| **pylint** | 3.0.3         | Static analysis        |
| **prettier** | 4.0.0-alpha.8 | Non-Python formatting |
| **pre-commit-hooks** | 4.5.0 | AST check, trailing whitespace, etc. |

### Running Linters

```bash
# Run all pre-commit hooks
pre-commit run --all-files

# Individual tools
black api/ *.py
isort api/ *.py
flake8 api/ *.py
pylint api/ *.py
```

### Linting Configuration

- **flake8** (`setup.cfg`): Ignores E501 (line length), E226, W503, E231
- **pylint** (`pyproject.toml`): Disables `import-error`, `line-too-long`, `invalid-name`, `protected-access`
- **isort**: Uses `black` profile for compatibility

## Testing

There is no pytest/unittest framework. Tests are executable scripts:

- `test_api.py` - Main API test with flags: `TESTAUTHENTICATE`, `TESTAPIMETHODS`, `TESTAPIENDPOINTS`, `TESTAPIFROMJSON`
- `test_c1000x_mqtt_controls.py` - C1000X MQTT control tests
- `examples/` directories contain JSON exports used for offline testing via `myapi.testDir(folder)`

```bash
# Run API tests (requires .env with credentials or env vars)
python test_api.py

# Run with example data (offline)
# Set TESTAPIFROMJSON = True in test_api.py
```

## Key Conventions

### Code Style
- **Black-formatted** code (no manual line length enforcement)
- **isort** for import ordering (black-compatible profile)
- All API methods are `async`
- Type hints used throughout
- Docstrings on public methods

### Naming
- Classes: `PascalCase` (e.g., `AnkerSolixApi`, `SolixMqttDevice`)
- Methods/functions: `snake_case`
- Constants/enums: `UPPER_CASE` or `PascalCase` for enum members
- Protected members start with `_` (pylint `protected-access` disabled)

### Error Handling
- Custom exceptions in `api/errors.py`: `AuthorizationError`, `ConnectError`, `RequestLimitError`, etc.
- Methods raise specific exceptions rather than generic ones

### Commit Messages
- Short, descriptive (e.g., "MQTT updates", "Various fixes", "Add basic Solarbank PPS system support")
- Feature branch PRs from contributors merged via merge commits

## Environment Variables

Scripts use `.env` files (via python-dotenv) for credentials:
- `ANKERUSER` - Anker account email
- `ANKERPASSWORD` - Anker account password
- `ANKERCOUNTRY` - Country code

## CI/CD

- No GitHub Actions workflows configured
- Pre-commit hooks enforce code quality locally
- Releases managed via `.github/release.yml` (changelog categories: `enhancement`, `bug`)
