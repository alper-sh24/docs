# Using UV and Fastapi

## Configuration

UV toml:

```toml
[project]
name = "wawi"
version = "0.1.0"
description = "Add your description here"
readme = "README.md"
requires-python = ">=3.12"
dependencies = [
    "aiohttp>=3.13.2",
    "fastapi>=0.124.2",
    "requests>=2.32.5",
    "uvicorn>=0.38.0",
]
```

Running the python server:

```
uv run uvicorn main:app --host 0.0.0.0 --reload --port 5005
```

## Daemonization

Creating a **my_fastapi.service** file in **/etc/systemd/system** directory:

```
[Unit]
Description=My FastAPI Server

[Service]
User=<username>
WorkingDirectory=/home/<username>/path/to/main.py
ExecStart=/home/<username>/.local/bin/uv run uvicorn main:app --host 0.0.0.0 --port 5008
Restart=always

[Install]
WantedBy=multi-user.target
```

- Since uv is not globally recognized, make sure to run `which uv` where you normally use it to find its original path, and **use the original path** in the service config.
- Port is up to you.
