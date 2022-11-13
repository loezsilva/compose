# Criar arquivos na VPS

    mkdir -p data/configurations
    touch docker-compose.yml
    touch data/traefik.yml
    touch data/acme.json
    touch data/configurations/dynamic.yml
    chmod 600 data/acme.json


# Docker Compose
Local do arquivo: <i>~/docker-compose.yml</i>

    version: '3.7'

    services:
      traefik:
        image: traefik:v2.4
        container_name: traefik
        restart: always
        security_opt:
          - no-new-privileges:true
        ports:
          - 80:80
          - 443:443
        volumes:
          - /etc/localtime:/etc/localtime:ro
          - /var/run/docker.sock:/var/run/docker.sock:ro
          - ./data/traefik.yml:/traefik.yml:ro
          - ./data/acme.json:/acme.json
          # Add folder with dynamic configuration yml
          - ./data/configurations:/configurations
        networks:
          - proxy
        labels:
          - "traefik.enable=true"
          - "traefik.docker.network=proxy"
          - "traefik.http.routers.traefik-secure.entrypoints=websecure"
          - "traefik.http.routers.traefik-secure.rule=Host(`traefik.domain`)"
          - "traefik.http.routers.traefik-secure.middlewares=user-auth@file"
          - "traefik.http.routers.traefik-secure.service=api@internal"

    networks:
      proxy:
        external: true

    
# Configuração estática
Local do arquivo: <i>~/data/traefik.yml</i><br/>

    api:
      dashboard: true

    entryPoints:
      web:
        address: :80
        http:
          redirections:
            entryPoint:
              to: websecure

      websecure:
        address: :443
        http:
          middlewares:
            - secureHeaders@file
            - nofloc@file
          tls:
            certResolver: letsencrypt

    pilot:
      dashboard: false

    providers:
      docker:
        endpoint: "unix:///var/run/docker.sock"
        exposedByDefault: false
      file:
        filename: /configurations/dynamic.yml

    certificatesResolvers:
      letsencrypt:
        acme:
          email: admin@domain
          storage: acme.json
          keyType: EC384
          httpChallenge:
            entryPoint: web

      buypass:
        acme:
          email: admin@domain
          storage: acme.json
          caServer: https://api.buypass.com/acme/directory
          keyType: EC256
          httpChallenge:
            entryPoint: web

        

# Configuração dinâmica
Local do arquivo: <i>~/data/configurations/dynamic.yml</i><br/><br/>

    http:
      middlewares:
        nofloc:
          headers:
            customResponseHeaders:
              Permissions-Policy: "interest-cohort=()"
        secureHeaders:
          headers:
            sslRedirect: true
            forceSTSHeader: true
            stsIncludeSubdomains: true
            stsPreload: true
            stsSeconds: 31536000   

        # UserName : admin
        # Password : qwer1234    
        user-auth:
          basicAuth:
            users:
              - "admin:$apr1$tm53ra6x$FntXd6jcvxYM/YH0P2hcc1"

    tls:
      options:
        default:
          cipherSuites:
            - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
            - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
            - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
            - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
            - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
            - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
          minVersion: VersionTLS12

# APP compose

    version: '3'

    services:
      django:
        build: .
        expose:
          - 8000
        container_name: django-app
        command:
          - bash
          - -c
          - |
            python manage.py migrate
            python manage.py runserver 0.0.0.0:8000
        volumes:
          - .:/app
        networks:
          - proxy
          - default
        ports:
          - "8000:8000"

        labels:
          - "traefik.enable=true"
          - "traefik.docker.network=proxy"
          - "traefik.http.routers.app-secure.entrypoints=websecure"
          - "traefik.http.routers.app-secure.rule=Host(`domain`)"
          # - "traefik.http.routers.app-secure.service=wordpress-service"
          # - "traefik.http.services.app-service.loadbalancer.server.port=8000"

    networks:
      proxy:
        external: true
