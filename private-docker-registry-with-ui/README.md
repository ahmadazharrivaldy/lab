# Setup Private Docker Registry with UI

## Install Docker Engine
```
sudo apt-get update && sudo apt install -y ca-certificates curl gnupg
```
```
{
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
}
```
```
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```
sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
```
sudo docker run hello-world
```

## Install Private Docker Registry

```
cat > registry-compose.yml <<EOF
version: '3'
services:
#Registry
  registry-server:
    image: registry:2
    restart: always
    ports:
    - "5000:5000"
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: basic-realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/htpasswd
      PATH: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
      ## allow registry ui to manage registry
      REGISTRY_STORAGE_DELETE_ENABLED: 'true'
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Origin: '["*"]'
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Methods: '[HEAD,GET,OPTIONS,DELETE]'
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Credentials: '[true]'
      REGISTRY_HTTP_HEADERS_Access-Control-Allow-Headers: '[Authorization,Accept,Cache-Control]'
      REGISTRY_HTTP_HEADERS_Access-Control-Expose-Headers: '[Docker-Content-Digest]'
    volumes:
      - registrydata:/data
      - ./registry-config/credentials.yml:/etc/docker/registry/config.yml
      - ./registry-config:/auth
      # bind path, so image still persists
      - /data/_data:/var/lib/registry
    networks:
      - mynet
    container_name: registry-server
  registry-ui:
    image: joxit/docker-registry-ui:latest
    restart: always
    ports:
      - 8080:80
    environment:
      - SINGLE_REGISTRY=true
      - REGISTRY_TITLE=Private Docker Registry
      - NGINX_PROXY_PASS_URL=http://rz-private-registry:5000
      - SHOW_CONTENT_DIGEST=true
      - SHOW_CATALOG_NB_TAGS=true
      - CATALOG_MIN_BRANCHES=1
      - CATALOG_MAX_BRANCHES=1
      - TAGLIST_PAGE_SIZE=100
      - REGISTRY_SECURED=false
      - CATALOG_ELEMENTS_LIMIT=1000
      ## registry-ui will create delete image button
      - DELETE_IMAGES=true
    depends_on:
      - registry-server
    networks:
      - mynet
    container_name: registry-ui
# Docker Networks
networks:
  mynet:
    driver: bridge
# Volumes
volumes:
  registrydata:
    driver: local
EOF
```
```
mkdir registry-config
```
```
cat > ~/registry-config/credentials.yml <<EOF
version: 0.1
log:
  fields:
    service: registry
storage:
  delete:
    enabled: true
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
    Access-Control-Allow-Origin: ['*']
    Access-Control-Allow-Methods: ['HEAD', 'GET', 'OPTIONS', 'DELETE']
    Access-Control-Allow-Headers: ['Authorization', 'Accept', 'Cache-Control']
    Access-Control-Max-Age: [1728000]
    Access-Control-Allow-Credentials: [true]
    Access-Control-Expose-Headers: ['Docker-Content-Digest']
auth:
  htpasswd:
    realm: basic-realm
    path: /auth/htpasswd
EOF
```
```
sudo docker compose -f registry-compose.yaml up -d
```
