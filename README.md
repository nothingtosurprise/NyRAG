# nyrag

A simple tool for crawling web pages and documents, converting them to markdown format for RAG applications.

## Features

- 🌐 Crawl websites and extract content
- 📄 Process local documents (PDF, DOCX, etc.)
- 📝 Convert HTML and documents to markdown using markitdown
- 💾 Store raw markdown files on disk
- ⚙️ YAML-based configuration
- 🚀 Simple CLI workflow: `nyrag --config path/to/config.yml`

## Installation

```bash
pip install nyrag
```

For development:
```bash
git clone https://github.com/abhishekkrthakur/nyrag.git
cd nyrag
pip install -e .
```

## Usage

Create a configuration file and run:

```bash
nyrag --config path/to/config.yml
```

### Configuration File

**Web Crawling:**
```yaml
name: web-rag
mode: web
start_loc: https://example.com/
exclude:
  - https://example.com/admin/*
  - https://example.com/private/*

# Optional crawl parameters
crawl_params:
  respect_robots_txt: true      # Respect robots.txt (default: true)
  aggressive_crawl: false       # Faster crawling (default: false)
  follow_subdomains: true       # Follow subdomains (default: true)
  strict_mode: false            # Only crawl URLs matching start pattern (default: false)
  user_agent_type: chrome       # chrome, firefox, safari, mobile, bot (default: chrome)
  # custom_user_agent: "..."    # Custom user agent (optional)
  # allowed_domains:            # Explicitly allowed domains (optional)
  #   - example.com

rag_params:
  embedding_model: sentence-transformers/all-MiniLM-L6-v2
  chunk_size: 500
  chunk_overlap: 50
```

**Document Processing:**
```yaml
name: doc-rag
mode: docs
start_loc: /path/to/documents/
exclude:
  - "*.csv"
  - /path/to/documents/exclude_this.docx

# Optional document processing parameters
doc_params:
  recursive: true               # Process subdirectories (default: true)
  include_hidden: false         # Include hidden files (default: false)
  follow_symlinks: false        # Follow symbolic links (default: false)
  max_file_size_mb: 100         # Max file size in MB (optional)
  file_extensions:              # Only process these types (optional)
    - .pdf
    - .docx
    - .txt

rag_params:
  embedding_model: sentence-transformers/all-MiniLM-L6-v2
  chunk_size: 500
  chunk_overlap: 50
```

### Crawl Parameters (Web Mode)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `respect_robots_txt` | bool | `true` | Respect robots.txt rules |
| `aggressive_crawl` | bool | `false` | Enable aggressive crawling (higher speed, more concurrent requests) |
| `follow_subdomains` | bool | `true` | Follow links to subdomains of the start URL |
| `strict_mode` | bool | `false` | Only crawl URLs that match the start URL pattern |
| `user_agent_type` | str | `chrome` | User agent type: `chrome`, `firefox`, `safari`, `mobile`, `bot` |
| `custom_user_agent` | str | `None` | Custom user agent string (overrides `user_agent_type`) |
| `allowed_domains` | list | `None` | Explicitly allowed domains (auto-detected from `start_loc` if not set) |

### Doc Parameters (Docs Mode)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `recursive` | bool | `true` | Process subdirectories recursively |
| `include_hidden` | bool | `false` | Include hidden files (starting with `.`) |
| `follow_symlinks` | bool | `false` | Follow symbolic links |
| `max_file_size_mb` | float | `None` | Maximum file size in MB (files larger will be skipped) |
| `file_extensions` | list | `None` | Only process files with these extensions (e.g., `[".pdf", ".docx"]`) |

### Output

All processed data will be saved in `nyrag-{name}/` folder:

```bash
nyrag --config configs/example.yml
# Creates: ./nyrag-web-rag/
```

## Examples

See the `configs/example.yml` file for a complete example with both web and docs mode configurations.

Run the example:
```bash
nyrag --config configs/example.yml
```

## API Server (Optional)

Run the FastAPI server:

```bash
uvicorn nyrag.api:app --host 0.0.0.0 --port 8000
```

### Chat UI with Vespa Cloud

After deploying + feeding to Vespa Cloud, run the chat UI against your Cloud endpoint:

```bash
# Use the same config you deployed (for schema name + embedding model)
export NYRAG_CONFIG=configs/example.yml

# Your mTLS endpoint (from `vespa_cloud.get_mtls_endpoint()` / deploy output)
export VESPA_URL="https://<your-endpoint>.z.vespa-app.cloud"
# VESPA_PORT is optional (defaults to 443 for Vespa Cloud)

# Data-plane mTLS credentials (created by Vespa CLI / pyvespa)
# Defaults to $HOME/.vespa/devrel-public.nyrag<projectname>.default; set these if your files live elsewhere.
export VESPA_CLIENT_CERT="$HOME/.vespa/<tenant>.<application>.<instance>/data-plane-public-cert.pem"
export VESPA_CLIENT_KEY="$HOME/.vespa/<tenant>.<application>.<instance>/data-plane-private-key.pem"

# LLM (required for chat answers)
export OPENROUTER_API_KEY=...
# export OPENROUTER_MODEL=openai/gpt-5.1-codex-max

uvicorn nyrag.api:app --host 0.0.0.0 --port 8000
```

Open `http://localhost:8000/chat`.

## Vespa backend

`nyrag` uses Vespa via `pyvespa`. It can deploy for you using either:

- **Local dev (default):** `VespaDocker` (starts a local Vespa container and deploys the generated app package)
- **Production:** `VespaCloud` (deploys to Vespa Cloud)

### Local (VespaDocker)

```bash
export NYRAG_LOCAL=1
export NYRAG_VESPA_DOCKER_IMAGE=vespaengine/vespa:latest  # optional

nyrag --config path/to/config.yml
```

### Vespa Cloud (VespaCloud)

```bash
export NYRAG_LOCAL=0
export VESPA_CLOUD_TENANT=your-tenant
# Optional (defaults to the generated app package name, e.g. `nyrag<projectname>`)
# export VESPA_CLOUD_APPLICATION=your-app
export VESPA_CLOUD_INSTANCE=default  # optional

# One of these (depending on your pyvespa setup):
export VESPA_CLOUD_API_KEY_PATH=/path/to/api-key.pem
# export VESPA_CLOUD_API_KEY="-----BEGIN PRIVATE KEY-----..."

# mTLS credentials for feeding/querying (typical for Vespa Cloud):
export VESPA_CLIENT_CERT=/path/to/vespa-cert.pem
export VESPA_CLIENT_KEY=/path/to/vespa-key.pem

nyrag --config path/to/config.yml
```

### Bring your own Vespa (skip deploy)

```bash
export NYRAG_VESPA_DEPLOY=0
export VESPA_URL=http://localhost
# VESPA_PORT is optional (defaults to 8080 for local)
nyrag --config path/to/config.yml
```
