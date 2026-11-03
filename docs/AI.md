# Modular Self-Hosted Web Application Architecture

**Role:** Senior Solutions Architect and Technical Mentor  
**Project:** Modular, Docker-based web application  
**Primary technologies:** Nginx, Docker, Python, optional PHP

---

## 1. Project Objective

Build a modular, self-hosted web application using Nginx as the central reverse proxy.

The application consists of independent services:

| URL | Service | Responsibility |
|---|---|---|
| `/` | Dashboard | Central landing page |
| `/journal/` | Journal | Daily notes and journal entries |
| `/health/` | Health | Fitness and health tracking |
| `/finance/` | Finance | Future financial tracking |

The architectural objectives are:

- Centralized HTTP/HTTPS access through Nginx.
- Independent application containers.
- Independent service deployment and upgrades.
- Shared external user experience.
- Isolated application responsibilities.
- Persistent application data.
- Ability to introduce additional technologies without redesigning the infrastructure.

## 2. Proposed Architecture

```text
                 Web Browser
                      |
                HTTPS / HTTP
                      |
                      v
              +---------------+
              |     NGINX     |
              | Reverse Proxy |
              +---------------+
                      |
               Docker Network
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
     Dashboard     Journal      Health
      Static       Python       Python
      HTML         FastAPI      FastAPI
          |           |           |
          v           v           v
       Files       Database    Database
````

Nginx acts as the single public entry point.

The browser accesses the application through one domain:

text

```text
https://myapp.example.com/
https://myapp.example.com/journal/
https://myapp.example.com/health/
https://myapp.example.com/finance/
```

Backend services communicate over a private Docker network.

Architectural principle: Nginx knows where services are. Services should not need to know where other services are.

## 3. Nginx Reverse Proxy

Nginx is responsible for:

- Accepting incoming HTTP/HTTPS connections.
- TLS termination.
- URL-based routing.
- Forwarding requests to backend containers.
- Adding forwarding headers.
- Serving static content where appropriate.
- Handling backend connection failures.

### 3.1 Why use a reverse proxy?

Without Nginx, individual services might be exposed through separate ports:

text

```text
myapp.example.com:8001
myapp.example.com:8002
myapp.example.com:8003
```

With Nginx:

text

```text
myapp.example.com/          -> Dashboard
myapp.example.com/journal/  -> Journal
myapp.example.com/health/   -> Health
myapp.example.com/finance/  -> Finance
```

Only Nginx needs publicly accessible ports.

### 3.2 Nginx routing configuration

Example:

nginx

```nginx
server {
    listen 80;
    server_name myapp.example.com;

    location = /journal {
        return 301 /journal/;
    }

    location /journal/ {
        proxy_pass http://journal:8000/;
    }

    location = /health {
        return 301 /health/;
    }

    location /health/ {
        proxy_pass http://health:8000/;
    }

    location / {
        proxy_pass http://dashboard:8080;
    }
}
```

Docker resolves service names such as `journal` and `health` through its internal DNS.

### 3.3 URI rewriting and trailing slashes

The trailing slash in `proxy_pass` affects the URI forwarded to the backend.

With:

nginx

```nginx
location /journal/ {
    proxy_pass http://journal:8000/;
}
```

The matching `/journal/` prefix is removed.

| Browser request | Backend receives |
| --- | --- |
| `/journal/` | `/` |
| `/journal/new` | `/new` |
| `/journal/entries/123` | `/entries/123` |

Without the trailing slash:

nginx

```nginx
location /journal/ {
    proxy_pass http://journal:8000;
}
```

The original URI is preserved.

| Browser request | Backend receives |
| --- | --- |
| `/journal/` | `/journal/` |
| `/journal/new` | `/journal/new` |

For this architecture, stripping the external service prefix is recommended.

Each application can then operate independently from its own root URL.

Explicit redirects should normalize URLs such as `/journal` to `/journal/`.

### 3.4 Forwarded headers

Nginx should forward information about the original request.

Example:

nginx

```nginx
location /journal/ {
    proxy_pass http://journal:8000/;

    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Prefix /journal;
}
```

| Header | Purpose |
| --- | --- |
| Host | Original hostname |
| X-Real-IP | Original client IP |
| X-Forwarded-For | Client/proxy IP chain |
| X-Forwarded-Proto | Original HTTP/HTTPS scheme |
| X-Forwarded-Host | Original hostname |
| X-Forwarded-Prefix | External application prefix |

Only trust forwarded headers from known proxies.

### 3.5 Docker networking

Create a dedicated Docker network:

yaml

```yaml
networks:
  web-network:
    driver: bridge
```

Attach the containers to this network.

Nginx can communicate with the journal application using:

text

```text
http://journal:8000
```

It does not need to know the container's IP address.

Use `expose` for internal service ports:

yaml

```yaml
journal:
  build: ./journal
  expose:
    - "8000"
  networks:
    - web-network
```

Avoid publishing backend ports unnecessarily:

yaml

```yaml
ports:
  - "8000:8000"
```

Only Nginx should normally publish ports 80 and 443.

Docker DNS consideration: Nginx may retain an old resolved container IP after a backend is recreated. For an initial deployment, reloading Nginx after backend recreation is reasonable. Dynamic DNS resolution can be introduced later.

### 3.6 Should Nginx run in its own container?

Yes.

Advantages:

- Independent lifecycle.
- Independent upgrades.
- Centralized routing configuration.
- Consistent external access.
- Application services can restart independently.

### 3.7 HTTPS and TLS

Nginx should terminate HTTPS.

text

```text
Browser
   |
   | HTTPS
   v
Nginx
   |
   | HTTP over private Docker network
   v
Backend
```

Certificate management options:

- Let's Encrypt.
- Certbot.
- DNS-based ACME challenges.

Example HTTPS configuration:

nginx

```nginx
server {
    listen 443 ssl;
    server_name myapp.example.com;

    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    location /journal/ {
        proxy_pass http://journal:8000/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /health/ {
        proxy_pass http://health:8000/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        proxy_pass http://dashboard:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Redirect HTTP to HTTPS:

nginx

```nginx
server {
    listen 80;
    server_name myapp.example.com;

    return 301 https://$host$request_uri;
}
```

Certificates must be stored persistently and private keys protected.

### 3.8 Backend availability

If the journal service stops, Nginx will normally return a `502 Bad Gateway` or `504 Gateway Timeout`.

Configure reasonable timeouts:

nginx

```nginx
location /journal/ {
    proxy_pass http://journal:8000/;

    proxy_connect_timeout 3s;
    proxy_send_timeout 30s;
    proxy_read_timeout 30s;

    proxy_intercept_errors on;
    error_page 502 503 504 = /service-unavailable.html;
}
```

Docker restart policies can help recover failed containers.

Health checks should be used to determine application readiness.

A failure in one application should not affect unrelated services.

Avoid blindly retrying write operations, since retries can produce duplicate records.

## 4. PHP: Architectural Assessment

PHP is optional.

It should not be introduced merely because it is available.

### 4.1 PHP-FPM

A typical PHP deployment uses:

- Nginx as the HTTP server.
- PHP-FPM as the PHP process manager.
- PHP files as the application.

Communication:

text

```text
Browser
   |
   v
Nginx
   |
   | FastCGI
   v
PHP-FPM
   |
   v
PHP Application
```

Nginx communicates with PHP-FPM using:

nginx

```nginx
fastcgi_pass php:9000;
```

This differs from Python applications, which normally communicate through HTTP:

nginx

```nginx
proxy_pass http://journal:8000;
```

### 4.2 Possible PHP roles

Option A: PHP as the dashboard

PHP handles the central dashboard while Python handles Journal and Health.

Advantages:

- Server-rendered dashboard.
- Independent Python applications.
- Opportunity to use a PHP framework such as Laravel.

Disadvantages:

- Additional runtime.
- Additional deployment complexity.
- Another technology to maintain.

Option B: PHP as the primary application

PHP owns the main application and integrates with the Python services.

This is appropriate if there is a substantial requirement for a PHP-based application.

It is unnecessary if the Python applications already provide their own interfaces.

Option C: No PHP initially

Use static HTML/CSS/JavaScript for the dashboard and Python for application services.

Advantages:

- Fewer technologies.
- Fewer containers.
- Less operational overhead.
- No PHP-FPM configuration.

Disadvantage:

- Dynamic dashboard functionality eventually requires JavaScript or a backend.

### 4.3 Recommendation

Do not introduce PHP initially.

Use:

| Component | Technology |
| --- | --- |
| Reverse proxy | Nginx |
| Dashboard | Static HTML/CSS/JavaScript |
| Journal | Python |
| Health | Python |
| Finance | Python or another suitable runtime |
| Database | SQLite initially; PostgreSQL when justified |

PHP can be introduced later without changing the fundamental architecture.

## 5. Python Application Architecture

Python is suitable for the independent application services.

Recommended frameworks:

- FastAPI: API-oriented applications.
- Flask: Lightweight web applications.
- Django: More comprehensive application framework.

### 5.1 Example Journal structure

text

```text
journal/
│
├── Dockerfile
├── requirements.txt
│
├── app/
│   ├── main.py
│   ├── routes/
│   │   ├── entries.py
│   │   └── search.py
│   ├── models/
│   │   └── entry.py
│   ├── services/
│   │   └── journal_service.py
│   └── templates/
│       ├── index.html
│       └── entry.html
│
└── data/
```

### 5.2 Example FastAPI application

python

```python
from fastapi import FastAPI

app = FastAPI(
    title="Journal Service"
)

@app.get("/")
def index():
    return {
        "application": "Journal",
        "status": "running"
    }

@app.get("/entries")
def get_entries():
    return {
        "entries": []
    }

@app.get("/health")
def health_check():
    return {
        "status": "healthy"
    }
```

Run with Uvicorn:

bash

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

The application listens on the container network interface.

Nginx handles external HTTP and HTTPS connections.

For applications mounted under URL prefixes, configure framework root-path handling and generated URLs appropriately.

### 5.3 Independent application responsibilities

| Journal | Health |
| --- | --- |
| Journal entries | Workouts |
| Daily notes | Exercise history |
| Categories | Measurements |
| Search | Fitness statistics |
| Attachments | Activity summaries |

Each application should own its own data.

Avoid having one application's code directly access another application's database.

If communication is necessary, use APIs.

text

```text
Journal -----> Journal Database

Health ------> Health Database

Dashboard ---> Journal API
           |
           +--> Health API
```

This minimizes coupling and makes individual services replaceable.

## 6. Docker Compose

A starting Compose configuration:

yaml

```yaml
services:

  nginx:
    image: nginx:stable
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/certs:/etc/nginx/certs:ro
      - ./nginx/html:/usr/share/nginx/html:ro
    networks:
      - web-network
    restart: unless-stopped

  journal:
    build: ./journal
    container_name: journal
    expose:
      - "8000"
    volumes:
      - journal-data:/app/data
    networks:
      - web-network
    restart: unless-stopped

  health:
    build: ./health
    container_name: health
    expose:
      - "8000"
    volumes:
      - health-data:/app/data
    networks:
      - web-network
    restart: unless-stopped

  dashboard:
    image: nginx:stable
    container_name: dashboard
    volumes:
      - ./dashboard:/usr/share/nginx/html:ro
    expose:
      - "80"
    networks:
      - web-network
    restart: unless-stopped

networks:
  web-network:
    driver: bridge

volumes:
  journal-data:
  health-data:
```

The dashboard has its own Nginx container serving static files.

The central Nginx container routes requests to it.

### 6.1 Why use named volumes?

Example:

yaml

```yaml
volumes:
  - journal-data:/app/data
```

The journal container can be replaced without deleting its persistent data.

Containers should generally be disposable.

Application code, configuration, and persistent data should be managed separately.

Backups must be implemented independently of Docker volumes.

## 7. URL Prefix Considerations

Path-based routing introduces an important application-development requirement.

Suppose the journal application contains:

HTMLWrap linesCopy codePreview

```html
<link rel="stylesheet" href="/static/style.css">
```

The browser requests:

text

```text
myapp.example.com/static/style.css
```

But the intended resource is:

text

```text
myapp.example.com/journal/static/style.css
```

Solutions include:

1. Configure applications to generate URLs with their external prefixes.
2. Use relative URLs where appropriate.
3. Configure Nginx to route static resources correctly.
4. Configure framework root-path support.

For this project, retain path prefixes.

Ensure that the following work correctly:

- Login redirects.
- Form submissions.
- JavaScript API requests.
- Images and uploaded files.
- CSS and JavaScript resources.
- WebSocket connections if introduced.

## 8. Architectural Trade-offs

| Decision | Recommendation | Reason |
| --- | --- | --- |
| Reverse proxy | Nginx | Central routing and TLS |
| Proxy container | Separate | Independent lifecycle |
| Dashboard | Static HTML | Minimal complexity |
| Journal | Python | Independent application |
| Health | Python | Independent application |
| PHP | Optional | Introduce only when justified |
| Networking | Docker bridge | Simple internal communication |
| Database | SQLite initially | Minimal overhead |
| Database evolution | PostgreSQL | More advanced capabilities |
| Deployment | Docker Compose | Straightforward operations |
| HTTPS | Let's Encrypt | Automated certificates |
| Authentication | Centralized later | Avoid duplicated login systems |

### Authentication

Eventually, a unified login should support:

text

```text
/
 /journal/
 /health/
 /finance/
```

Avoid implementing four separate authentication systems.

A future identity provider supporting OpenID Connect (OIDC) could provide centralized authentication.

This does not need to be implemented immediately.

However, the applications should be designed so that centralized authentication can be introduced later.

## 9. Implementation Roadmap

### Phase 1 — Infrastructure foundation

- Install Docker and Docker Compose.
- Create the project directory.
- Create the Docker network.
- Deploy Nginx independently.
- Verify access through the reverse proxy.

### Phase 2 — Dashboard

- Create a simple HTML landing page.
- Add navigation to Journal and Health.
- Configure Nginx to serve the dashboard.
- Verify the root URL.

### Phase 3 — Journal service

- Create a Python application.
- Build its Docker image.
- Connect it to the Docker network.
- Configure `/journal/` routing.
- Add persistent storage.

### Phase 4 — Health service

- Create a second independent Python application.
- Configure `/health/` routing.
- Give it independent persistent storage.
- Verify independent lifecycle management.

### Phase 5 — HTTPS and operational reliability

- Configure TLS certificates.
- Implement health checks.
- Configure service-unavailable pages.
- Implement backups.
- Review container security and permissions.

### Phase 6 — Application integration

- Consider centralized authentication.
- Introduce dashboard data aggregation.
- Evaluate whether PHP provides value.
- Add additional services such as Finance.

## 10. Final Recommended Technology Stack

| Component | Technology | Status |
| --- | --- | --- |
| Reverse proxy | Nginx | Core |
| Containerization | Docker | Core |
| Orchestration | Docker Compose | Core |
| Dashboard | HTML/CSS/JavaScript | Core |
| Journal | Python + FastAPI | Core |
| Health | Python + FastAPI | Core |
| Database | SQLite initially | Core |
| Database evolution | PostgreSQL | Future |
| PHP-FPM | PHP | Deferred |
| TLS | Let's Encrypt | Core for public HTTPS |

### Core architectural principle

Each application should be:

- Independently deployable.
- Independently maintainable.
- Responsible for its own data.
- Accessible through a well-defined interface.
- Replaceable without redesigning unrelated services.

Nginx provides the unified external interface.

Docker provides process isolation and networking.

Python provides application functionality.

The dashboard provides a cohesive user experience.

PHP remains an optional technology that can be introduced when a genuine requirement justifies it.

## 11. First Implementation Milestone

Begin with infrastructure rather than application functionality.

The first milestone should deliver:

- A Docker Compose project.
- A dedicated Nginx container.
- A shared Docker network.
- A static dashboard container.
- A placeholder Journal container.
- A placeholder Health container.
- Working routing for `/`, `/journal/`, and `/health/`.

Then test the architecture by independently stopping, starting, and restarting containers.

The goal is to demonstrate that the architecture works as a modular platform before adding application complexity.

### Environment decision

Before implementation, establish the deployment environment:

- Linux server (Ubuntu or Debian).
- NAS (such as Synology).
- Docker Desktop.

This affects filesystem mounts, networking, permissions, and certificate management.



