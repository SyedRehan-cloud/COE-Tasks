_format_version: "3.0"

services:

  # =========================================================
  # 1. PATH-BASED ROUTING (/users)
  # =========================================================
  - name: user-service
    url: http://127.0.0.1:5678
    routes:
      - name: user-path-route
        paths:
          - /users
        strip_path: true

        annotations:
          explanation: >
            When request comes to /users:
            1. Kong matches path "/users"
            2. Route is selected
            3. strip_path=true removes /users before forwarding
            4. Upstream receives "/"
            Example:
              /users → http://127.0.0.1:5678/


  # =========================================================
  # 2. HOST-BASED ROUTING (users.local)
  # =========================================================
  - name: host-user-service
    url: http://127.0.0.1:5678
    routes:
      - name: user-host-route
        hosts:
          - users.local
        paths:
          - /
        strip_path: false

        annotations:
          explanation: >
            When request comes:
              Host: users.local
            Kong checks HOST first.
            If host matches → route selected.
            Path "/" is secondary here.
            No strip_path → full path forwarded.
            Example:
              users.local:8000 → upstream service


  # =========================================================
  # 3. METHOD-BASED ROUTING (GET only)
  # =========================================================
  - name: method-service
    url: http://127.0.0.1:5678
    routes:
      - name: method-route
        paths:
          - /method
        methods:
          - GET
        strip_path: true

        annotations:
          explanation: >
            Kong matches BOTH:
              1. Path = /method
              2. HTTP method = GET

            Behavior:
              GET /method  → ALLOWED
              POST /method → NOT MATCHED

            Then strip_path removes /method before forwarding.


  # =========================================================
  # 4. HEADER-BASED ROUTING (version=v1)
  # =========================================================
  - name: header-service
    url: http://127.0.0.1:5678
    routes:
      - name: header-route
        paths:
          - /versioned
        headers:
          version:
            - v1
        strip_path: true

        annotations:
          explanation: >
            Kong checks:
              1. Path = /versioned
              2. Header version = v1

            Only requests like:
              /versioned + version:v1

            will match this route.

            Without header → route is skipped.


  # =========================================================
  # 5. QUERY PARAM ROUTING (?type=admin)
  # =========================================================
  - name: query-service
    url: http://127.0.0.1:5678
    routes:
      - name: query-route
        paths:
          - /query
        strip_path: true

        annotations:
          explanation: >
            Kong route matching is ONLY path-based here.
            Query params are NOT used in routing decision.

            Example:
              /query?type=admin

            Kong forwards entire query string to backend:
              /query?type=admin → upstream


  # =========================================================
  # 6. REGEX PATH ROUTING (/users/{id})
  # =========================================================
  - name: regex-service
    url: http://127.0.0.1:5678
    routes:
      - name: regex-route
        paths:
          - ~/users/\d+
        strip_path: false

        annotations:
          explanation: >
            Regex routing enables dynamic path matching.

            Matches:
              /users/123 ✔
              /users/999 ✔
              /users/abc ✘

            Because:
              \d+ = only numbers

            Full path is forwarded as-is.


  # =========================================================
  # 7. MULTI PATH ROUTING (/orders, /purchases)
  # =========================================================
  - name: multi-path-service
    url: http://127.0.0.1:5678
    routes:
      - name: multi-path-route
        paths:
          - /orders
          - /purchases
        strip_path: true

        annotations:
          explanation: >
            One service can handle multiple entry points.

            Matches:
              /orders → service
              /purchases → service

            Both routes forward to same backend.

            strip_path=true removes prefix before forwarding.


  # =========================================================
  # 8. DEFAULT FALLBACK ROUTE (/)
  # =========================================================
  - name: fallback-service
    url: http://127.0.0.1:5678
    routes:
      - name: fallback-route
        paths:
          - /
        strip_path: false

        annotations:
          explanation: >
            This acts as catch-all route.

            If no other route matches:
              → this route is used

            Example:
              /random-api → fallback-service

            IMPORTANT:
              Should always be last logical safety route.
