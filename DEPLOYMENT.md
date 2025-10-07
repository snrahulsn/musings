# Deployment Guide

This guide covers deploying the Workflow Analytics App to a new server.

## Prerequisites on Target Server

- Python 3.10 or higher
- Docker and Docker Compose
- At least 2GB RAM
- Ubuntu/Debian or similar Linux distribution

## Deployment Steps

### 1. Transfer Archive

Transfer the `pulse_revamp.tar.gz` file to your server:

```bash
scp pulse_revamp.tar.gz user@your-server:/home/user/
```

### 2. Extract on Server

```bash
cd /home/user/
tar -xzf pulse_revamp.tar.gz
cd pulse_revamp
```

### 3. Configure Environment

```bash
cp .env.example .env
nano .env  # or vim/vi
```

**Required configurations:**
- `LLM_API_KEY`: Your xAI Grok or OpenAI API key
- `LLM_API_BASE`: API endpoint (default: https://api.x.ai/v1)
- `DATABASE_URL`: Should work as-is (postgresql://workflow_user:workflow_pass@localhost:5433/workflow_analytics)

### 4. Set Up Python Environment

```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

**Note:** The requirements.txt has Meltano commented out. Uncomment when ready to set up ETL pipelines.

### 5. Install LangChain Dependencies

```bash
pip install langchain==0.3.27 langchain-community==0.3.30 langchain-openai==0.3.35
```

### 6. Start PostgreSQL

```bash
docker-compose up -d
```

Wait 10 seconds for PostgreSQL to initialize, then verify:

```bash
docker-compose ps
```

### 7. Initialize Database

```bash
python init_db.py
```

You should see: "Database initialized successfully!"

### 8. Start Services

**Option A: Using Screen (Recommended for long-running sessions)**

```bash
# Start Flask API
screen -dmS flask bash -c "cd /home/user/pulse_revamp && source venv/bin/activate && export PYTHONPATH=/home/user/pulse_revamp && python app/api/routes.py > logs/flask.log 2>&1"

# Start Streamlit
screen -dmS streamlit bash -c "cd /home/user/pulse_revamp && source venv/bin/activate && export PYTHONPATH=/home/user/pulse_revamp && streamlit run app/ui/main_app.py --server.port 8501 --server.headless true > logs/streamlit.log 2>&1"

# View sessions
screen -ls

# Attach to session
screen -r flask   # or screen -r streamlit
# Detach: Ctrl+A then D
```

**Option B: Using systemd (Production)**

Create service files for Flask and Streamlit (see systemd section below).

**Option C: Simple Background Processes**

```bash
# Start Flask
cd /home/user/pulse_revamp
source venv/bin/activate
export PYTHONPATH=/home/user/pulse_revamp
nohup python app/api/routes.py > logs/flask.log 2>&1 &

# Start Streamlit
nohup streamlit run app/ui/main_app.py --server.port 8501 --server.headless true > logs/streamlit.log 2>&1 &
```

### 9. Verify Services

```bash
# Check Flask API
curl http://localhost:5001/health

# Check Streamlit (should return HTML)
curl -I http://localhost:8501

# View logs
tail -f logs/flask.log
tail -f logs/streamlit.log
```

### 10. Configure Firewall (if needed)

```bash
# Allow ports
sudo ufw allow 5001/tcp  # Flask API
sudo ufw allow 8501/tcp  # Streamlit UI
sudo ufw reload
```

### 11. Access Application

- **Local access**: http://localhost:8501
- **Remote access**: http://your-server-ip:8501

**Security Note:** For production, use a reverse proxy (Nginx) with SSL.

## Systemd Service Files (Production)

### Flask API Service

Create `/etc/systemd/system/pulse-api.service`:

```ini
[Unit]
Description=Pulse Analytics Flask API
After=network.target docker.service
Requires=docker.service

[Service]
Type=simple
User=your-user
WorkingDirectory=/home/user/pulse_revamp
Environment="PYTHONPATH=/home/user/pulse_revamp"
Environment="PATH=/home/user/pulse_revamp/venv/bin"
ExecStart=/home/user/pulse_revamp/venv/bin/python app/api/routes.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Streamlit Service

Create `/etc/systemd/system/pulse-streamlit.service`:

```ini
[Unit]
Description=Pulse Analytics Streamlit UI
After=network.target pulse-api.service
Requires=pulse-api.service

[Service]
Type=simple
User=your-user
WorkingDirectory=/home/user/pulse_revamp
Environment="PYTHONPATH=/home/user/pulse_revamp"
Environment="PATH=/home/user/pulse_revamp/venv/bin"
ExecStart=/home/user/pulse_revamp/venv/bin/streamlit run app/ui/main_app.py --server.port 8501 --server.headless true
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Enable and start:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable pulse-api pulse-streamlit
sudo systemctl start pulse-api pulse-streamlit

# Check status
sudo systemctl status pulse-api
sudo systemctl status pulse-streamlit

# View logs
sudo journalctl -u pulse-api -f
sudo journalctl -u pulse-streamlit -f
```

## Nginx Reverse Proxy (Optional but Recommended)

Install Nginx:

```bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx
```

Create `/etc/nginx/sites-available/pulse`:

```nginx
server {
    listen 80;
    server_name your-domain.com;

    # Streamlit UI
    location / {
        proxy_pass http://localhost:8501;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
    }

    # Flask API
    location /api {
        proxy_pass http://localhost:5001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable site:

```bash
sudo ln -s /etc/nginx/sites-available/pulse /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

Add SSL with Let's Encrypt:

```bash
sudo certbot --nginx -d your-domain.com
```

## Stopping Services

### Screen sessions:
```bash
screen -S flask -X quit
screen -S streamlit -X quit
```

### Background processes:
```bash
pkill -f "app/api/routes.py"
pkill -f "streamlit run"
```

### Systemd:
```bash
sudo systemctl stop pulse-api pulse-streamlit
```

### Docker:
```bash
docker-compose down
```

## Updating the Application

1. Stop services
2. Backup database:
   ```bash
   docker exec pulse_revamp-postgres-1 pg_dump -U workflow_user workflow_analytics > backup.sql
   ```
3. Extract new tar.gz over existing directory
4. Run migrations (if any)
5. Restart services

## Troubleshooting

### Port conflicts:
```bash
# Check what's using ports
sudo lsof -i :5001
sudo lsof -i :8501

# Change ports in .env and docker-compose.yml
```

### Database connection issues:
```bash
# Check PostgreSQL logs
docker-compose logs postgres

# Restart PostgreSQL
docker-compose restart postgres
```

### Module import errors:
Always ensure PYTHONPATH is set:
```bash
export PYTHONPATH=/home/user/pulse_revamp
```

### View all logs:
```bash
tail -f logs/*.log
```

## Archive Contents

The tar.gz excludes:
- `venv/` (virtual environment - recreate on server)
- `__pycache__/` (Python cache)
- `.git/` (git repository)
- `logs/*.log` (log files)
- `data/*` (database data)

All configuration files, source code, and dependencies list are included.
