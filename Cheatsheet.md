==================== POWERSHELL ====================

ssh -i "$env:USERPROFILE\Downloads\s20230204038" s20230204038@187.52.122.100


==================== CLONE REPOSITORY ====================

cd /home/s20230204038

git clone https://github.com/S-Arshad032/cse3100-sticky-note-previous-year.git

cd cse3100-sticky-note-previous-year

cat .env.example


==================== MYSQL CONTAINER ====================

sudo docker start bookdb

sudo docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' bookdb

COPY THE PRINTED IP.


==================== MYSQL USER ====================

sudo docker exec -it bookdb mysql -uroot -prootpass

CREATE USER IF NOT EXISTS 's20230204038'@'%' IDENTIFIED BY 'PASSWORD';

ALTER USER 's20230204038'@'%' IDENTIFIED BY 'PASSWORD';

GRANT SELECT ON bookdb.* TO 's20230204038'@'%';

FLUSH PRIVILEGES;

EXIT;


==================== CREATE .env ====================

cp .env.example /home/s20230204038/.env

nano /home/s20230204038/.env


PASTE THESE VALUES INSIDE .env:

PORT=3038
HOST=127.0.0.1
DB_HOST=172.17.0.2
DB_PORT=3306
DB_NAME=bookdb
DB_USER=s20230204038
DB_PASSWORD=PASSWORD


REPLACE PASTE_PRINTED_CONTAINER_IP WITH THE ACTUAL IP.

SAVE NANO: Ctrl+O, Enter
EXIT NANO: Ctrl+X


==================== PM2 ====================

cd /home/s20230204038/cse3100-sticky-note-previous-year

pm2 start server.js --name bookapi-038

pm2 save

pm2 startup


COPY AND RUN THE EXACT sudo env PATH=... COMMAND PRINTED BY PM2.

THEN RUN:

pm2 save


==================== LOCAL HEALTH CHECK ====================

curl http://127.0.0.1:3038/api/health


EXPECTED:

{"status":"ok","database":"up"}


==================== NGINX ====================

sudo nano /etc/nginx/sites-available/s20230204038.conf


PASTE THIS INSIDE NGINX FILE:

server {
    listen 80;
    server_name s20230204038.austattendance.online;

    location / {
        proxy_pass http://127.0.0.1:3038;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}


or:

server {
    listen 80;
    server_name s20230204038.austattendance.online;

    location / {
        proxy_pass http://127.0.0.1:3038;
    }
}


SAVE NANO: Ctrl+O, Enter
EXIT NANO: Ctrl+X


==================== ENABLE NGINX ====================

sudo ln -sf /etc/nginx/sites-available/s20230204038.conf /etc/nginx/sites-enabled/s20230204038.conf

sudo nginx -t && sudo systemctl reload nginx


==================== FINAL CHECK ====================

curl http://s20230204038.austattendance.online/api/health


OPEN IN BROWSER:

http://s20230204038.austattendance.online

http://s20230204038.austattendance.online/api/health


EXPECTED HEALTH:

{"status":"ok","database":"up"}


IMPORTANT:

DO NOT RUN npm install.
DO NOT RUN npm run build.
DO NOT RUN PM2 WITH sudo.
DO NOT DROP OR DELETE bookdb.
DO NOT STOP OR REMOVE THE bookdb CONTAINER.