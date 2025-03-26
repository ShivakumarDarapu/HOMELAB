# HOMELAB SETUP

![alt text](homelab.jpg)

# THINGS TO DO WHILE SETTING UP NEW APP

### 1. CREATE DNS ENTRY IN CLOUDFLARE 

- DECIDE UPON THE DNS NAME AND CREATE A CNAME RECORD FOR THE DNS NAME THAT MAPS TO THE PROXY STARTING WITH 7.. SO THAT THE TRAFFIC IS ROUTED TO THE CLOUDFLARE SERVICE RUNNING ON SERVER 192.168.1.25.

### 2. CLOUDFLARED LOCAL CONFIG TO ROUTE TRAFFIC

- ADD DNS ENTRY IN CONFIG FILE LOCATED ON PATH /home/shiv/.cloudflared/config.yaml SO THAT THE TRAFFIC NOW COMMING FROM CLOUDFLARE KNOWS WHERE TO GO IN LOCAL NETWORK.

- ONCE DONE, RUN tunnel down & then tunnelup commands which is an alias created in 192.168.1.25 to start the cloud flare tunnel 

``` yaml
- alias tunnelup='cloudflared tunnel run <tunnel-id>
```

```yaml
- alias tunneldown='kill $(ps aux | grep "cloudflared tunnel run <tunnel-id> | awk '{print $2})
```
### 3. DNS ENTRY IN BIND9

- MAKE APPROPRIATE DNS ENTRY IN BIND9 CONFIG LOCATED IN PATH - /home/shiv/Bind9/config/test-potatohead-live.zone

- once done run command
``` command
docker compose down and docker compose up -d
```

### 3. TRAEFIK CONFIGURATION

### 3.1 FOR APPLICATION HOSTED ON DOCKER

- IF YOU ARE DEPLOYING APPLICATION ON DOCKER NETWORK ADD THE BELOW LABELS SO THAT TRAFIK CAN EXPOSE THE SAME, GET THE CERTIFICATES USING DNS CHALLENGE AND ALSO CREATE ROUTES SO IT CAN DIRECT THE TRAFFIC TO YOUR POD.

``` YAML
labels:
      - "traefik.enable=true" # to let trafik know that this needs to be exposed
      # HTTP Router
      # - "traefik.http.routers.portainer.rule=Host(`containers.potatohead.live`)"
      # - "traefik.http.routers.portainer.entrypoints=web"
      # - "traefik.http.services.portainer.loadbalancer.server.port=80"

      # HTTPS Router
      - "traefik.http.routers.<route-name>.rule=Host(`containers.potatohead.live`)"
      - "traefik.http.routers.<route-name>.entrypoints=websecure"
      - "traefik.http.routers.<route-name>.tls=true"
      - "traefik.http.routers.<route-name>.tls.certresolver=cloudflare"
      - "traefik.http.routers.<route-name>.service=<route-name>-svc"
      - "traefik.http.services.<route-name>-svc.loadbalancer.server.port=<port-no-on-which-your-application-listens-nonhttps-port>"

```
- CURRENTLY THE TRAEFIK IS ONLY CONNECTED TO TWO NETWORKS ONE ON BACKEND AND THE OTHER ON FRONTEND, MAKE SURE YOU CHOOSE ANY ONE OF THEM ELSE IT WONT WORK. 

``` YAML
    networks:
      - backend
      - frontend
networks:
  frontend:
    external: true  
  backend:
    external: true

```
- ALSO MAKE SURE THE ENTRY IN BIND9 AND CLOUDFLARED CONFIG IS MADE TO REDIRECT TRAFFIC TO TRAEFIK LB HOSTED ON 1921.68.1.25

- RUN COMMAND
``` COMMAND
docker compose down && docker compose up -d
```

- RUN COMMAND TO CHECK IF THE CERTIFICATE IS ISSUED FOR THE NEW DNS BY GOING TO THE FOLDER /home/shiv/Docker-Files/Traefik/

``` YAML
    cat certs/cloudflare-acme.json | jq . | grep "<your dns>"
```

### 3.2 FOR APPLICATION HOSTED ON K3S OR VM

- MAKE SURE THE ENTRY IN BIND9 AND CLOUDFLARED CONFIG IS MADE TO REDIRECT TRAFFIC TO TRAEFIK LB HOSTED ON 1921.68.1.25.

- MAKE APPROPRIATE CHANGES IN CONFIG FILE LOCATED ON PATH /home/shiv/Docker-Files/Traefik/conf/traefik_dynamic.yaml.

``` YAML
http:
  routers:
    portainer-router-https:
      rule: "Host(`test.dev.live`)"
      entryPoints:
        - websecure
      service: portainer
      tls:
        certResolver: cloudflare
    portainer-router-http:
      rule: "Host(`test.dev.live`)"
      entryPoints:
        - web
      middlewares:
        - redirect-to-https
      service: portainer
  middlewares:
    secureHeaders:
      headers:
        sslRedirect: true
        stsSeconds: 63072000
        stsIncludeSubdomains: true
        stsPreload: true
        forceSTSHeader: true
  
  services:
    portainer:
      loadBalancer:
        servers:
          - url: "https://test.dev.live:9443"

```
- RUN COMMAND
``` COMMAND
docker compose down && docker compose up -d
```

TROUBLESHOOTING:-

1. CHECK IF THE PROXY IS ENABLED ON CLOUDFLARE FOR YOUR DNS IN THE DNS CONFIG TAB.
2. CHECK IN CLIENT WHERE YOU RAN THE TUNNELUP COMMAND TO SEE IF WHEN U HIT THE WEB PAGE YOU SEE ANY LOG / ERROR. IF U SEE ERROR OR LOG THIS MEANS THE TRAFFIC FROM INTERNET --> CLOUDFLARE --> TUNNEL --> IS REACHING YOUR SERVER.
3. CHECK LOGS OF TRAEFIK IF YOU SEE ANY LOGS, IF NOT THEN THE ROUTING IN CLOUDFLARED CONFIG IS INCORRECT PERHAPS. TRY MANUALLY DOING A CURL FROM THE SERVER AND SEE IF THE PAGE IS LOADING, IF ITS NOT ITS THE DOCKER CONTAINER THAT HAS SOME ISSUES( TO PERFORM THIS TEST EXPOSE THE PORTS TO SOME NODE PORTS AND TEST).
4. IF YOU SEE SOMETHING LIKE BELOW ERROR 

``` C++
ERR  error="Unable to reach the origin service. The service may be down or it may not be responding to traffic from cloudflared: tls: failed to verify certificate: x509: cannot validate certificate for 192.168.1.25 because it doesn't contain any IP SANs" connIndex=0 event=1 ingressRule=0 originService=https://192.168.1.25:443
2025-03-25T12:53:00Z ERR Request failed error="Unable to reach the origin service. 
```
, NOTE THAT CURRENLY YOUR DOMAIN ON CLOUDFLARE DOESNT SUPPORT PROVIDING VALID CERTIFICATES FOR CLILD DOMAIN.
EXAMPLE:
``` YAML
PARENT_DOMAIN --> *.GOODLUCK.LIVE
CHILD_DOMAIN --> *.DEV.GOODLUCK.LIVE
```
YOU CAN VERIFY THIS WHEN U CREATE THE SAME DNS ENTRY IN CLOUDFLARE, U WILL SEE A WARNING LIKE ICON NEXT TO IT , IF ITS A CLILD DOMAIN.
