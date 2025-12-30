# Installing n8n on Ubuntu with free SSL

## Step 1: Config ufw
```shell
sudo ufw allow http
sudo ufw allow https
sudo ufw enable
```

## Step 2: Docker and Compose Setup
```shell
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

## Step 3: Install the Docker packages
```shell
apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
apt-cache policy docker-ce
apt install docker-ce
systemctl status docker
```

## Step 4: Creating a docker directory & yml file
```shell
cd
mkdir -p docker/n8n && cd docker/n8n
nano compose.yml
```

## Step 5: Add to compose.yml
```shell
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./certbot/www:/var/www/certbot
      - ./certbot/conf:/etc/letsencrypt
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - n8n
    restart: always

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - ./certbot/www:/var/www/certbot
      - ./certbot/conf:/etc/letsencrypt
    entrypoint: >
      sh -c "trap exit TERM; while :; do
        certbot renew --webroot -w /var/www/certbot --quiet;
        sleep 12h & wait $${!};
      done"

  n8n:
    image: n8nio/n8n
    volumes:
      - n8n_data:/home/node/.n8n
    environment:
      - N8N_HOST=<Your Domain>
    restart: always


volumes:
  n8n_data:
```

## Step 6: start nginx and n8n
```shell
docker compose up nginx -d
```

## Step 7: Nginx Initial Configuration
```shell
cd nginx/conf.d
sudo nano default.conf
```

## Step 8: Add to default.conf
```shell
server {
    listen 80;
    server_name <Your Domain>;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        proxy_pass http://n8n:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
    }
}
```

## Step 9: restart the nginx service
```shell
docker compose restart nginx
```
you should see the following Image
![](http://www.majidkalantarii.ir/github/secure_cookie.png)

## Step 10: Generating Certificates using Certbot
```shell
sudo docker compose run --rm --entrypoint certbot certbot \
  certonly --webroot -w /var/www/certbot \
  -d <Your Domain>
```

## Step 11: Nginx Final Configuration over HTTPS
Replace the contents of default.conf
```shell
server {
    listen 80;
    server_name <Your Domain>;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}


server {
    listen 443 ssl;
    server_name n8n.rcsm.ir;


    ssl_certificate /etc/letsencrypt/live/n8n.rcsm.ir/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.rcsm.ir/privkey.pem;

    location / {
        proxy_pass http://n8n:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
    }
}
```

## Step 12: Now restart nginx
```shell
docker compose restart nginx
```
