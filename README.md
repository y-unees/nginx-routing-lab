# The Nginx Routing Lab: Mastering Request Handlers

> A focused deep-dive into how Nginx resolves URIs using `root`, `index`, and `try_files` directives.

---

## Project Scope

This project is a practical reference guide for understanding how Nginx processes incoming requests and maps them to filesystem resources.

**Objective:**
- Build intuition for Nginx request resolution
- Compare directory-based vs explicit routing
- Solve real-world routing edge cases (notably `403 Forbidden`)

**Live Demo Environment:**
```
http://lab.unishdhungana.com.np
```
> Note: This domain is used for demonstration and may be inactive post-lab.

---

## The Lab Configuration

```nginx
server {
    listen 80;
    server_name lab.unishdhungana.com.np;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    # Directory-based routing
    location /abc/ {
        try_files $uri $uri/ =404;
    }

    # Explicit file mapping
    location /xyz/ {
        try_files /xyz.html =404;
    }

    location /pqr/ {
        try_files /random_text.html =404;
    }
}
```

---

## How Nginx Resolves Requests (Mental Model)

1. Match the best `location` block  
2. Combine with `root` to form a filesystem path  
3. Apply `index` if the target is a directory  
4. Evaluate `try_files` in order  
5. Return the first valid match or fallback  

---

## Pattern A: Standard Directory Indexing (`/abc/`)

### Behavior

Request:
```
/abc/
```

Nginx resolves:
```
/var/www/html/abc/index.html
```

---

### Result

![Directory Indexing Result](./images/abc-index.png)

---

### Multi-File Test (Key Validation)

Additional file:
```
/var/www/html/abc/test.html
```

Accessible via:
```
/abc/test.html
```

---

### Result

![Multi-file Test Result](./images/abc-test-file.png)

---

### Key Insight

This confirms that directory routing:
- Serves `index.html` by default  
- Exposes all files within the directory  
- Validates correct folder-wide mapping  

---

## Pattern B: Explicit File Mapping (`/xyz/` and `/pqr/`)

### Scenario

Map clean URIs to files that do not match the request path.

Example:
```
/xyz/ → xyz.html
/pqr/ → random_text.html
```

---

### Implementation

```nginx
location /xyz/ {
    try_files /xyz.html =404;
}

location /pqr/ {
    try_files /random_text.html =404;
}
```

---

### Result

![Explicit Routing Result](./images/pqr-custom.png)

---

### Key Insight

- Enables clean URL design  
- Removes dependency on directory structure  
- Provides deterministic file resolution  

---

## Pattern C: Solving the `403 Forbidden` Issue

### The Problem

Using `alias` resulted in:
```
403 Forbidden
```

---

### Root Cause

- Nginx expected a valid directory or index file  
- `alias` introduced ambiguity in file resolution  
- No fallback mechanism was defined  

---

### Incorrect Approach

```nginx
location /pqr/ {
    alias /var/www/html/random_text.html;
}
```

---

### Correct Approach

```nginx
location /pqr/ {
    try_files /random_text.html =404;
}
```

---

### Why `try_files` Works

- Provides explicit file resolution  
- Eliminates ambiguity  
- Ensures predictable behavior  

---

### Takeaway

| Method      | Behavior                | Result        |
|-------------|------------------------|---------------|
| `alias`     | Path substitution      | 403 errors    |
| `try_files` | Deterministic routing  | Reliable      |

---

## Configuration Workflow

```bash
# Validate configuration
sudo nginx -t

# Reload safely
sudo systemctl reload nginx

# Restart if required
sudo systemctl restart nginx
```

---

## Repository Structure

```text
.
├── .gitignore
├── README.md
├── configs/
│   ├── nginx.conf
│   └── routing.conf
├── web_root(/var/www/html)/
│   ├── index.html
│   ├── abc/ (index.html, test.html)
│   ├── xyz/ (xyz.html)
│   └── pqr/ (random_text.html)
└── images/
    ├── abc-index.png
    ├── abc-test-file.png
    └── pqr-custom.png
```

---

## What This Lab Demonstrates

- Accurate understanding of Nginx request resolution  
- Directory vs explicit routing patterns  
- Real-world debugging of 403 errors  
- Clean and reusable routing configurations  

---

## Key Takeaways

- Use `root + index` for directory-based routing  
- Use `try_files` for explicit and reliable mapping  
- Avoid `alias` unless fully necessary  
- Always validate configuration before applying  

---

This project is intentionally focused on routing clarity and serves as a reusable reference for developers working with Nginx.