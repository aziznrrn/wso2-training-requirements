# Brainmatics's WSO2 API Management Training Software Requirement

1. Buat instance server di IDCloudhost. Simpan username, password, dan ip server

2. Buka windows terminal

3. SSH ke server

    Jalankan perintah berikut

    ```powershell
    ssh <username>@<ip_server>
    ```

4. Install docker

    Jalankan perintah berikut

    ```bash
    curl -fsSL https://get.docker.com -o get-docker.sh
    sudo sh ./get-docker.sh --dry-run
    sudo apt install -y docker-compose
    export DOCKER_HOST=tcp://<ip_server>:2375
    ```

    Apabila docker belum bisa digunakan, mungkin perlu direstart sebelum docker bisa digunakan, jalankan perintah berikut untuk merestart server

    ```bash
    sudo init 6
    ```

    Anda akan keluar dari SSH, setelah beberapa saat, SSH ke server lagi seperti pada poin 3

5. Buat berkas-berkas konfigurasi

    Jalankan perintah-perintah berikut

    ```bash
    tee docker-compose-apim.yml <<EOF
    version: '3'

    services:
      api-manager:
        image: wso2/wso2am:4.0.0
        container_name: api-manager
        ports:
          - "8280:8280"
          - "8243:8243"
          - "9443:9443"
        depends_on:
          - mysql-db
        environment:
          - WSO2_USER=wso2user
          - WSO2_PASSWORD=wso2pass
          - WSO2_DATABASE_HOST=mysql-db
          - WSO2_DATABASE_PORT=3306
          - WSO2_DATABASE_NAME=wso2amdb
          - WSO2_DATABASE_USER=wso2amdbuser
          - WSO2_DATABASE_PASSWORD=wso2amdbpass

      mysql-db:
        image: mysql:5.7
        container_name: mysql-db
        environment:
          MYSQL_ROOT_PASSWORD: rootpass
          MYSQL_USER: wso2amdbuser
          MYSQL_PASSWORD: wso2amdbpass
          MYSQL_DATABASE: wso2amdb

      phpmyadmin:
        image: phpmyadmin/phpmyadmin
        container_name: phpmyadmin
        ports:
          - "8080:80"
        depends_on:
          - mysql-db
        environment:
          - PMA_HOST=mysql-db
          - PMA_PORT=3306
          - PMA_ARBITRARY=1
    EOF
    ```

    ```bash
    tee docker-compose-is.yml <<EOF
    version: '3'

    services:
      identity-server:
        image: wso2/wso2is:5.7.0
        container_name: is
        ports:
          - "9550:9443"
        depends_on:
          - mysql-db
        environment:
          - WSO2_USER=admin
          - WSO2_PASSWORD=admin
          - WSO2_MYSQL_HOST=mysql-db
          - WSO2_MYSQL_PORT=3306
          - WSO2_MYSQL_USERNAME=root
          - WSO2_MYSQL_PASSWORD=rootpass

      mysql-db:
        image: mysql:5.7
        container_name: mysql-db
        environment:
          MYSQL_ROOT_PASSWORD: rootpass
          MYSQL_DATABASE: identitydb
          MYSQL_USER: identityuser
          MYSQL_PASSWORD: identitypass
    EOF
    ```

    ```bash
    tee docker-compose-mi.yml <<EOF
    version: '3'

    services:
      micro-integrator:
        image: wso2/micro-integrator
        container_name: micro-integrator
        ports:
          - "8290:8290"
          - "8253:8253"
        depends_on:
          - mysql-db
        environment:
          - MI_CONFIG_ENV=/home/wso2carbon/wso2mi-1.2.0/repository/conf/deployment.toml

      mysql-db:
        image: mysql:5.7
        container_name: mysql-db-mi
        environment:
          MYSQL_ROOT_PASSWORD: rootpass
          MYSQL_DATABASE: midb
          MYSQL_USER: miuser
          MYSQL_PASSWORD: mipass
    EOF
    ```

    Script didapat dari [medium.com/@dikkumburage](https://medium.com/@dikkumburage/wso2-installation-using-docker-compose-b91585fdff4a)

6. Jalankan WSO2

    Jalankan perintah-perintah berikut

    ```bash
    sudo docker-compose -f docker-compose-apim.yml up -d
    ```

    ```bash
    sudo docker-compose -f docker-compose-is.yml up -d
    ```

    ```bash
    sudo docker-compose -f docker-compose-mi.yml up -d
    ```

7. Install NGINX

    Jalankan perintah berikut

    ```bash
    sudo apt install -y nginx
    sudo systemctl enable nginx
    sudo systemctl start nginx
    ```

8. Buat file konfigurasi NGINX

    generate self-signed certificate

    ```bash
    sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/self-signed.key -out /etc/nginx/self-signed.crt
    ```

    membuat file konfigurasi

    ```bash
    sudo tee /etc/nginx/sites-available/wso2_services <<EOF
    server {
        listen 10443 ssl;
        server_name <ip_server>;

        ssl_certificate /etc/nginx/self-signed.crt;
        ssl_certificate_key /etc/nginx/self-signed.key;

        location / {
            proxy_pass https://localhost:9443;  # API Manager
            proxy_set_header Host \$host;
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto \$scheme;
        }
    }
    server {
        listen 10443 ssl;
        server_name 103.181.143.19;

        ssl_certificate /etc/nginx/self-signed.crt;
        ssl_certificate_key /etc/nginx/self-signed.key;

        location / {
            proxy_pass http://localhost:9443;  # API Manager
            proxy_set_header Host \$host;
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto \$scheme;
            proxy_set_header X-Forwarded-Port 10443;  # Add this line
        }
    }

    server {
        listen 10550 ssl;
        server_name <ip_server>;

        ssl_certificate /etc/nginx/self-signed.crt;
        ssl_certificate_key /etc/nginx/self-signed.key;

        location / {
            proxy_pass https://localhost:9550;  # WSO2 IS
            proxy_set_header Host \$host;
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto \$scheme;
        }
    }

    server {
        listen 10290 ssl;
        server_name <ip_server>;

        ssl_certificate /etc/nginx/self-signed.crt;
        ssl_certificate_key /etc/nginx/self-signed.key;

        location / {
            proxy_pass http://localhost:8290;  # Micro Integrator
            proxy_set_header Host \$host;
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto \$scheme;
        }
    }
    ```

    nonaktifkan `listen [::]:80 default_server;` pada `/etc/nginx/sites-enabled/default`

    ```bash
    sudo sed -i 's/^listen \[::\]:80 default_server;/#&/' /etc/nginx/sites-enabled/default
    ```

    buat simbolic link konfigurasi nginx

    ```bash
    sudo ln -s /etc/nginx/sites-available/wso2_services /etc/nginx/sites-enabled/
    ```

    restart nginx

    ```bash
    sudo systemctl restart nginx
    ```

9. Allow request ke port-port WSO2

    ```bash
    sudo ufw allow 10443/tcp
    sudo ufw allow 10550/tcp
    sudo ufw allow 10290/tcp
    sudo ufw enable
    ```

10. WSO2 dapat diakses pada `https://<ip_address>:10443`
