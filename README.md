# Two-Tier Flask + MySQL App with CI/CD Pipeline

## What is this project?

This is a two-tier web application. That means the app has two separate parts:
- **Tier 1** — A Flask web app (the part users see and interact with)
- **Tier 2** — A MySQL database (the part that stores messages)

Both parts run as separate Docker containers on an AWS EC2 server. Whenever I push new code to GitHub, GitHub Actions automatically builds the Docker image, pushes it to Docker Hub, and deploys it to EC2. This is called a **CI/CD pipeline** — it's how real companies ship code to production.

---

## Tools Used

| Tool | Why |
|------|-----|
| Python + Flask | Backend web app |
| MySQL | Database to store messages |
| Docker | Package the app into containers |
| AWS EC2 | Cloud server to host everything |
| GitHub Actions | Automatically build and deploy on every git push |
| Docker Hub | Store the built Docker image |
| GitHub | Store the code + trigger the pipeline |

---

## How the App Works

1. User opens the app in browser
2. Flask fetches all messages from MySQL and shows them
3. User types a message and clicks Send
4. jQuery sends the message to Flask without reloading the page (AJAX)
5. Flask saves the message to MySQL
6. Message appears on screen instantly

---

## Project Structure

```
two-tier-app/
├── app.py                        # Flask backend
├── Dockerfile                    # How to build the Flask container
├── requirements.txt              # Python dependencies
├── message.sql                   # SQL schema
├── .github/
│   └── workflows/
│       └── deploy.yaml           # GitHub Actions CI/CD pipeline
└── templates/
    └── index.html                # Frontend UI
```

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Developer Machine                            │
│                                                                     │
│   code change  ──►  git push origin main                           │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         GitHub                                      │
│                                                                     │
│   Repository  ──►  Triggers GitHub Actions on push to main         │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   GitHub Actions Runner                             │
│                                                                     │
│   1. Checkout code                                                  │
│   2. Setup Docker Buildx                                            │
│   3. Login to Docker Hub                                            │
│   4. Build Docker image  ──►  Push to Docker Hub                   │
│   5. SSH into EC2  ──►  Deploy containers                          │
└──────────────┬──────────────────────────┬───────────────────────────┘
               │                          │
               ▼                          ▼
┌──────────────────────┐    ┌─────────────────────────────────────────┐
│      Docker Hub      │    │              AWS EC2                    │
│                      │    │                                         │
│  murtaza007/         │    │  ┌──────────────────────────────────┐   │
│  webapp:latest  ◄────┼────┼─►│     Docker Network: two-tier     │   │
│                      │    │  │                                  │   │
└──────────────────────┘    │  │  ┌─────────────┐  ┌──────────┐  │   │
                            │  │  │   webapp    │  │ mysql-db │  │   │
                            │  │  │  (Flask)    │◄─►│ (MySQL)  │  │   │
                            │  │  │  port:5000  │  │ port:3306│  │   │
                            │  │  └──────┬──────┘  └──────────┘  │   │
                            │  │         │                        │   │
                            │  └─────────┼────────────────────────┘   │
                            │            │                             │
                            └────────────┼─────────────────────────────┘
                                         │
                                         ▼
                            ┌────────────────────────┐
                            │       Browser          │
                            │                        │
                            │  http://<ec2-ip>:5000  │
                            └────────────────────────┘
```

---

## CI/CD Flow (How Auto Deploy Works)

```
I push code to GitHub
        ↓
GitHub Actions triggers the pipeline
        ↓
Runner checks out the code
        ↓
Docker image is built and pushed to Docker Hub
        ↓
GitHub Actions SSHs into EC2
        ↓
EC2 pulls the latest image and restarts containers
        ↓
New version of app is live on EC2 at http://<ec2-ip>:5000
```

---

## GitHub Actions Pipeline Steps

```yaml
1. Checkout code
2. Setup Docker Buildx
3. Login to Docker Hub
4. Build and push Docker image to Docker Hub
5. SSH into EC2 and:
   - Create docker network (two-tier)
   - Pull and run mysql:8.0 container
   - Pull latest webapp image
   - Stop and remove old webapp container
   - Run new webapp container with correct env vars
```

---

## GitHub Secrets Required

| Secret | What it stores |
|--------|---------------|
| `DOCKER_USERNAME` | Docker Hub username |
| `DOCKER_PASSWORD` | Docker Hub password |
| `EC2_HOST` | EC2 public IP address |
| `EC2_SSH_KEY` | Private SSH key to access EC2 |
| `HOST_USERNAME` | EC2 login username (e.g. ubuntu) |
| `MYSQL_ROOT_PASSWORD` | MySQL root password |
| `MYSQL_DATABASE` | MySQL database name |
| `MYSQL_USER` | MySQL app user |
| `MYSQL_PASSWORD` | MySQL app user password |

---

## Issues I Faced & How I Fixed Them

### Issue 1 — Tried to install MySQL locally (not needed)
**What happened:** I started installing MySQL using Homebrew on my Mac thinking the app needed it locally.

**The loophole:** In a Dockerized project, MySQL runs as a container. You don't need to install it on your machine at all.

**Fix:** Stopped the Homebrew installation with `Ctrl+C` and used MySQL as a Docker container instead.

---

### Issue 2 — Dockerfile was using unnecessary multi-stage build
**What happened:** The Dockerfile had two stages — `base` and `production` — which added complexity for no reason.

**The loophole:** Multi-stage builds are useful when you're compiling code (like Go or Java). For a simple Python Flask app, one stage is enough.

**Fix:** Simplified the Dockerfile to a single stage.

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN apt-get update && apt-get install -y pkg-config default-libmysqlclient-dev build-essential
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

---

### Issue 3 — `pkg-config: not found` error during Docker build
**What happened:** When building the Docker image, pip failed to install `mysqlclient` with this error:
```
/bin/sh: 1: pkg-config: not found
Can not find valid pkg-config name.
```

**The loophole:** `python:3.12-slim` is a minimal image — it doesn't have system-level MySQL libraries that `mysqlclient` needs to compile itself.

**Fix:** Added these system packages in the Dockerfile before running pip install:
```dockerfile
RUN apt-get update && apt-get install -y \
    pkg-config \
    default-libmysqlclient-dev \
    build-essential \
    && rm -rf /var/lib/apt/lists/*
```

---

### Issue 4 — Flask crashed because MySQL wasn't ready yet
**What happened:** Flask container started and immediately tried to connect to MySQL, but MySQL was still initializing. Flask crashed with:
```
MySQLdb.OperationalError: (2002, "Can't connect to server on 'mysql-db' (115)")
```

**The loophole:** MySQL takes 20-30 seconds to fully initialize on first boot. Flask starts instantly and crashes before MySQL is ready.

**Fix:** Added a retry loop in `app.py` so Flask keeps retrying the DB connection instead of crashing:
```python
def init_db():
    for attempt in range(15):
        try:
            # connect and create table
            return
        except Exception as e:
            print(f"Retrying... {e}")
            time.sleep(5)
    raise Exception("Could not connect to the database after multiple attempts.")
```

---

### Issue 5 — `MYSQL_USER: default` caused MySQL error
**What happened:** MySQL refused to create the user because `default` is a reserved keyword in MySQL.

**The loophole:** You can't use reserved words as MySQL usernames.

**Fix:** Changed the username from `default` to `flaskuser`.

---

### Issue 6 — `debug=True` hardcoded in app.py
**What happened:** The Flask app had `debug=True` hardcoded, which is a security risk in production — it exposes system info to anyone who triggers an error.

**The loophole:** Debug mode should never be hardcoded. It should be controlled by an environment variable.

**Fix:**
```python
app.run(host='0.0.0.0', port=5000, debug=os.environ.get('FLASK_DEBUG', 'False') == 'True')
```

---

### Issue 7 — Wrong action path in deploy.yaml
**What happened:** The Docker login step had the wrong action path:
```yaml
uses: actions/docker/login@v2   # wrong
```

**The loophole:** `actions/docker/login` doesn't exist. The correct action is maintained by Docker, not GitHub.

**Fix:**
```yaml
uses: docker/login-action@v3    # correct
```

---

### Issue 8 — Entire `jobs` and `steps` were nested under `on` in deploy.yaml
**What happened:** The pipeline never ran correctly because the YAML structure was completely wrong — `jobs` and `steps` were indented under the `on` trigger block instead of being at the top level.

**The loophole:** YAML is indentation-sensitive. A single wrong indent level breaks the entire structure silently or causes unexpected behavior.

**Fix:** Restructured the file so `on`, `jobs`, and `steps` are all at their correct indentation levels.

---

### Issue 9 — `docker: command not found` on EC2
**What happened:** The GitHub Actions pipeline SSHed into EC2 and ran docker commands, but got:
```
bash: line 1: docker: command not found
Process exited with status 127
```

**The loophole:** A fresh EC2 instance doesn't have Docker installed. The pipeline assumes Docker is already there.

**Fix:** SSH into EC2 and install Docker manually:
```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
```
Then log out and back in for the group change to take effect.

---

### Issue 10 — Flask couldn't resolve MySQL hostname `mysql`
**What happened:** Flask kept failing with:
```
(2005, "Unknown server host 'mysql' (-3)")
```
Even though the MySQL container was running fine.

**The loophole:** The MySQL container was named `mysql-db`, but `app.py` defaults to connecting to a host named `mysql` when no `MYSQL_HOST` env var is passed. The webapp `docker run` command had no `-e` flags at all, so Flask used the hardcoded default.

**Fix:** Pass `MYSQL_HOST` and all other DB env vars explicitly when running the webapp container:
```bash
docker run --name webapp --network two-tier -p 5000:5000 --restart=always -d \
  -e MYSQL_HOST=mysql-db \
  -e MYSQL_USER=<secret> \
  -e MYSQL_PASSWORD=<secret> \
  -e MYSQL_DB=<secret> \
  <dockerhub-username>/webapp:latest
```

---

## How to Run Locally

```bash
# Clone the repo
git clone https://github.com/<your-username>/two-tier-app.git
cd two-tier-app

# Create the network
docker network create two-tier

# Run MySQL
docker run --name mysql-db \
  -e MYSQL_ROOT_PASSWORD=example \
  -e MYSQL_DATABASE=flask_db \
  -e MYSQL_USER=flaskuser \
  -e MYSQL_PASSWORD=flaskpassword \
  --network two-tier -p 3306:3306 -d mysql:8.0

# Run Flask app
docker run --name webapp --network two-tier -p 5000:5000 -d \
  -e MYSQL_HOST=mysql-db \
  -e MYSQL_USER=flaskuser \
  -e MYSQL_PASSWORD=flaskpassword \
  -e MYSQL_DB=flask_db \
  <your-dockerhub-username>/webapp:latest

# Open browser
http://localhost:5000
```

---

## Proof — App Running Live

### GitHub Actions Pipeline — Build Successful
![GitHub Actions Pipeline](github-action-pipeline.png)

### Flask App — Live on EC2
![Live App on EC2](live-app.png)

---

## Key Learnings

- Always pass env vars explicitly to every container that needs them — don't rely on defaults
- YAML is indentation-sensitive — one wrong indent breaks the entire pipeline
- A fresh EC2 instance has nothing pre-installed — always verify Docker is present before running the pipeline
- Container names on a Docker network act as hostnames — the name you give with `--name` is what other containers use to connect
- A retry loop in your app is a simple but powerful way to handle startup race conditions
- Never hardcode `debug=True` in production Flask apps
- Reserved words in MySQL (like `default`) will silently break your setup
- `docker/login-action` is maintained by Docker, not GitHub — always use the correct action namespace
