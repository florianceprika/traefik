# Traefik for Mark Shust's Docker Magento

## How to Use

### Setup Traefik

1. **Install mkcert:**
   - If you don't have it, install mkcert using Homebrew:
     ```sh
     brew install mkcert
     ```

2. **Install SSL Certificates:**
   - Install SSL certificates in `/docker/certs/`.
   - Use `mkcert` to set up your SSL certificates:
     ```sh
     mkcert -install
     mkcert -key-file docker/certs/nginx.key -cert-file docker/certs/nginx.crt localhost
     ```

3. **Create Docker Network:**
   - Create the required Docker network:
     ```sh
     docker network create webgateway
     ```

4. **Start Traefik:**
   - Once installed and network created, start Traefik with:
     ```sh
     docker compose up -d
     ```

5. **Access Traefik Dashboard:**
   - The Traefik dashboard should be available at: [https://dashboard.localhost/dashboard/#/http/routers](https://dashboard.localhost/dashboard/#/http/routers)

### Setup Magento

Now that you have Traefik running, it's time to add Traefik to your Magento project's `compose.yaml` file. Traefik will also allow you to run multiple Magento 2 instances simultaneously. It is not very stable yet, but it works. To manage multiple instances, you'll need to add a specific name to each instance. In this example, we will use colors, with "green" being the first instance.

1. **Add a Network:**
   - Add a network to all Docker services:
     ```yaml
     networks:
       - webgateway-green
     ```

   - Add the network configuration at the end of your `compose.yaml` file:
     ```yaml
     networks:
       webgateway-green:
         external: true
     ```

2. **Configure Services:**

   - **App Service:**
     ```yaml
     app:
       image: markoshust/magento-nginx:1.18-8
       networks:
         - webgateway-green
       volumes: &appvolumes
         - ~/.composer:/var/www/.composer:cached
         - ~/.ssh/id_rsa:/var/www/.ssh/id_rsa:cached
         - ~/.ssh/known_hosts:/var/www/.ssh/known_hosts:cached
         - appdata-green:/var/www/html
         - sockdata-green:/sock
         - ssldata-green:/etc/nginx/certs
       labels:
         - "traefik.enable=true"
         - "traefik.http.routers.green.tls=true"
         - "traefik.http.routers.green.entrypoints=websecure"
         - "traefik.http.services.green.loadbalancer.server.port=8443"
         - "traefik.http.services.green.loadbalancer.server.scheme=https"
     ```

   - **Mailcatcher Service:**
     ```yaml
     mailcatcher:
       image: sj26/mailcatcher
       labels:
         - "traefik.enable=true"
         - "traefik.docker.network=webgateway-green"
         - "traefik.http.routers.green_mailcatcher.tls=true"
         - "traefik.http.routers.green_mailcatcher.entrypoints=websecure"
         - "traefik.http.routers.green_mailcatcher.service=green_mailcatcher"
         - "traefik.http.services.green_mailcatcher.loadbalancer.server.port=1080"
       networks:
         - webgateway-green
     ```

   - **PhpMyAdmin Service:** (Note: This configuration is in the `compose.dev.yaml` file)
     ```yaml
     phpmyadmin:
       image: linuxserver/phpmyadmin
       env_file: env/db.env
       networks:
         - webgateway-green
       labels:
         - "traefik.enable=true"
         - "traefik.http.routers.green_phpmyadmin.tls=true"
         - "traefik.http.routers.green_phpmyadmin.entrypoints=websecure"
       depends_on:
         - db
     ```

   - **Database Service:**
     ```yaml
     db:
       image: mariadb:10.6
       command:
         --max_allowed_packet=64M
         --optimizer_use_condition_selectivity=1
         --optimizer_switch="rowid_filter=off"
       networks:
         - webgateway-green
       labels:
         - "traefik.enable=true"
         - "traefik.tcp.routers.db-green.rule=HostSNI(`*`)"
         - "traefik.tcp.services.db-green.loadbalancer.server.port=3306"
         - "traefik.tcp.routers.db-green.entrypoints=mysql"
       env_file: env/db.env
       volumes:
         - dbdata-green:/var/lib/mysql
     ```

     You can connect to the database using an external client like [TablePlus](https://tableplus.com) with the following configuration:
     - **Host:** 127.0.0.1
     - **Port:** 3306
     - **User:** root
     - **Password:** magento
     - **Database:** magento

You can find the full `compose.yaml` and `compose.dev.yaml` files in the `/compose` directory.