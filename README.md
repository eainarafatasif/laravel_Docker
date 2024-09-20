# laravel_Docker

To deploy a Laravel project on Docker using Nginx, MySQL, and an SSL certificate, follow these steps:

### 1. **Set Up Project Structure**
Create a folder for your project and include the following files:

```bash
your-laravel-project/
├── docker/
│   ├── nginx/
│   │   └── nginx.conf
│   ├── mysql/
│   ├── php/
│   └── ssl/
├── docker-compose.yml
└── .env
```

### 2. **Create Dockerfile for Laravel (PHP-FPM)**
Inside `docker/php/`, create a `Dockerfile` to define the PHP environment for Laravel.

```Dockerfile
FROM php:8.1-fpm

# Install necessary PHP extensions and system dependencies
RUN apt-get update && apt-get install -y \
    libfreetype6-dev libjpeg62-turbo-dev libpng-dev libonig-dev \
    libzip-dev zip unzip git curl \
    && docker-php-ext-install pdo pdo_mysql mbstring gd zip

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Set working directory
WORKDIR /var/www

# Copy Laravel project files
COPY . .

# Run composer install and optimize autoload
RUN composer install --optimize-autoloader --no-dev

# Set permissions
RUN chown -R www-data:www-data /var/www \
    && chmod -R 755 /var/www

EXPOSE 9000
CMD ["php-fpm"]
```

### 3. **Nginx Configuration**
Inside `docker/nginx/`, create the `nginx.conf` file to configure Nginx to serve your Laravel application.

```nginx
server {
    listen 80;
    server_name your-domain.com;

    root /var/www/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass php-fpm:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
    
    # Redirect HTTP to HTTPS
    listen 443 ssl;
    ssl_certificate /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;
}
```

### 4. **Set Up Docker Compose**
In the root of your project, create the `docker-compose.yml` file to orchestrate the services: Nginx, PHP-FPM, and MySQL.

```yaml
version: '3.8'

services:
  php-fpm:
    build:
      context: .
      dockerfile: docker/php/Dockerfile
    container_name: laravel-php
    working_dir: /var/www
    volumes:
      - ./:/var/www
    networks:
      - laravel-network

  nginx:
    image: nginx:latest
    container_name: laravel-nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./:/var/www
      - ./docker/nginx/nginx.conf:/etc/nginx/conf.d/default.conf
      - ./docker/ssl:/etc/nginx/ssl
    depends_on:
      - php-fpm
    networks:
      - laravel-network

  mysql:
    image: mysql:8.0
    container_name: laravel-mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: laravel_db
      MYSQL_USER: laravel_user
      MYSQL_PASSWORD: secret
    volumes:
      - ./docker/mysql:/var/lib/mysql
    ports:
      - "3306:3306"
    networks:
      - laravel-network

networks:
  laravel-network:
    driver: bridge
```

### 5. **Set Up SSL**
Generate SSL certificates and place them inside the `docker/ssl` directory:

```bash
mkdir -p docker/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout docker/ssl/server.key -out docker/ssl/server.crt
```

Make sure the certificate paths are correct in the Nginx configuration (`nginx.conf`).

### 6. **Environment Variables**
In your `.env` file (for Laravel and Docker):

```bash
APP_NAME=Laravel
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=mysql
DB_HOST=laravel-mysql
DB_PORT=3306
DB_DATABASE=laravel_db
DB_USERNAME=laravel_user
DB_PASSWORD=secret
```

### 7. **Running Docker Containers**
Now that your Docker setup is ready, run the following command to start your Docker containers:

```bash
docker-compose up --build -d
```

This will start Nginx, PHP-FPM, and MySQL services with the configuration provided.

### 8. **Laravel Application Setup**
Once the containers are running, you need to access the PHP container to set up Laravel:

```bash
docker exec -it laravel-php bash
```

Run the following inside the container:

```bash
php artisan key:generate
php artisan migrate
```

### 9. **Access Your Laravel App**
Your Laravel application should now be accessible via `http://localhost` or the domain you configured in your `nginx.conf`. If you set up SSL, access the app via `https://localhost` or your domain.

### 10. **(Optional) Update DNS and SSL for Production**
If deploying on a production server, make sure to point your domain to the server IP and acquire a real SSL certificate (e.g., from Let's Encrypt). You can modify your Nginx configuration to support automatic SSL certificate renewal using Certbot:

```bash
sudo certbot --nginx -d your-domain.com
```

---

With this setup, your Laravel project should be fully deployed on Docker with Nginx, MySQL, and SSL support. Let me know if you need help with any specific part!
