## 1. Creating the ASP.NET Web API Project

First, I created an **ASP.NET Web API** project using the .NET CLI or Visual Studio.

```powershell
# Using the .NET CLI
dotnet new webapi -n MyWebAPI
cd MyWebAPI
```

## 2. Adding MySQL and Entity Framework Core Migration

To integrate MySQL, I added the necessary **Entity Framework Core** packages:

```powershell
dotnet add package Pomelo.EntityFrameworkCore.MySql
dotnet add package Microsoft.EntityFrameworkCore.Design

```

Then, I configured **DbContext** inside `appsettings.json`:

```json
"ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=mydb;User=root;Password=mypassword;"
}

```

Next, I registered the `DbContext` in `Program.cs`:

```csharp
builder.Services.AddDbContext<MyDbContext>(options =>
    options.UseMySql(builder.Configuration.GetConnectionString("DefaultConnection"),
        ServerVersion.AutoDetect(builder.Configuration.GetConnectionString("DefaultConnection"))));

```

then i added the automatic Apply Migration when the project starts 

```csharp
var app = builder.Build();

WaitForDatabase(connectionString, maxRetries: 30, delayMilliseconds: 5000);

// Apply migrations only if needed
using (var scope = app.Services.CreateScope())
{
    var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();

    if (dbContext.Database.GetPendingMigrations().Any())
    {
        dbContext.Database.Migrate();
    }
}

private static void WaitForDatabase(string connectionString, int maxRetries, int delayMilliseconds)
{
    int retryCount = 0;
    while (retryCount < maxRetries)
    {
        try
        {
            using var connection = new MySqlConnection(connectionString);
            connection.Open();
            Console.WriteLine("Successfully connected to MySQL.");
            return;
        }
        catch (Exception ex)
        {
            retryCount++;
            Console.WriteLine($"Waiting for MySQL to be ready... Attempt {retryCount}/{maxRetries}: {ex.Message}");
            Thread.Sleep(delayMilliseconds);
        }
    }
    throw new Exception("Unable to connect to MySQL after multiple attempts.");
}

```

To apply the migrations, I used:

```
dotnet ef migrations add InitialCreate
dotnet ef database update

```

## 3. Configuring CORS

To allow cross-origin requests, I added the following to `Program.cs`:

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAllOrigins",
        policy =>
        {
            policy.AllowAnyOrigin()
                  .AllowAnyMethod()
                  .AllowAnyHeader();
        });
});

```

And enabled it:

```csharp
app.UseCors("AllowAllOrigins");

```

## 4. Adding Docker Support

I added **Docker support** using:

```powershell
dotnet add package Microsoft.NET.Build.Containers

```

Then, I created a **Dockerfile** outside the project directory:

```docker
# Base runtime image (ONLY .NET Runtime)
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

# Build stage (WITH .NET SDK)
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src

# Copy project files and restore dependencies
COPY ["DockerSSLWebAPI/DockerSSLWebAPI.csproj", "DockerSSLWebAPI/"]
RUN dotnet restore "./DockerSSLWebAPI/DockerSSLWebAPI.csproj"

# Copy the full source code
COPY . .

# Set working directory and build the app
WORKDIR "/src/DockerSSLWebAPI"
RUN dotnet build "./DockerSSLWebAPI.csproj" -c $BUILD_CONFIGURATION -o /app/build

# Publish the application
FROM build AS publish
RUN dotnet publish "./DockerSSLWebAPI.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

# Final runtime image (DO NOT INSTALL SDK HERE)
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish ./

# Run the Web API (No need for `dotnet-ef` here)
ENTRYPOINT ["dotnet", "DockerSSLWebAPI.dll"]

```

I then built the **Docker image**:

```powershell
docker build -t webapi:0.1 .

```

## 5. Creating `docker-compose.yml`

I added a **Docker Compose** file to run MySQL, the Web API, and Nginx:

```yaml
version: "3.8"

services:
  db:
    image: mysql:latest
    container_name: mysql_container
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: P@SS
      MYSQL_DATABASE: localdb
      MYSQL_USER: sa
      MYSQL_PASSWORD: P@SS
    ports:
      - "3307:3306"
    networks:
      - app_network
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -usa -pP@SS || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 15
      start_period: 70s
    volumes:
      - mysql_data:/var/lib/mysql

  webapi:
    build: .
    image: webapi:0.1
    container_name: webapi
    restart: always
    depends_on:
      db:
        condition: service_healthy
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=Server=db;Port=3306;Database=localdb;User=sa;Password=P@SS;
    networks:
      - app_network

  nginx:
    image: nginx:latest
    container_name: nginx_container
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - webapi
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    networks:
      - app_network

networks:
  app_network:

volumes:
  mysql_data:

```

## 6. Configuring Nginx Reverse Proxy

I created an `nginx.conf` file:

```json
events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    gzip  on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://webapi:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Port $server_port;
        }

        location /swagger/ {
            proxy_pass http://webapi:8080/swagger/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Port $server_port;
        }
    }
}

```

## 7. Opening Ports for External Access

I contacted my **internet provider** to open **ports 80 and 443**. If using a personal router, I set up **port forwarding** to allow external access to the local machine.

## 9. Running this CMD Sava

To start the containers, I ran:

```powershell
# Allow port 80 (HTTP)
New-NetFirewallRule -Name "Allow_HTTP_80" -DisplayName "Allow HTTP (80)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 80

# Allow port 443 (HTTPS)
New-NetFirewallRule -Name "Allow_HTTPS_443" -DisplayName "Allow HTTPS (443)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 443

# (Optional) Verify the rules have been created
Get-NetFirewallRule -Name "Allow_HTTP_80", "Allow_HTTPS_443"

```

## 8. Running the Setup

To start the containers, I ran:

```
docker-compose up -d

```

Now, users can access the API using:

```
http://myip:80/swagger

```

This setup allows a fully containerized ASP.NET Web API with **MySQL and Nginx reverse proxy** accessible from external devices.

+

## 10. Getting Certbot SSL

modify this and add the certbot 

```yaml
version: "3.8"

services:
  db:
    image: mysql:latest
    container_name: mysql_container
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: P@SS
      MYSQL_DATABASE: localdb
      MYSQL_USER: sa
      MYSQL_PASSWORD: P@SS
    ports:
      - "3307:3306"
    networks:
      - app_network
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h localhost -usa -pP@SS || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 15
      start_period: 70s
    volumes:
      - mysql_data:/var/lib/mysql

  webapi:
    build: .
    image: webapi:0.1
    container_name: webapi
    restart: always
    depends_on:
      db:
        condition: service_healthy
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=Server=db;Port=3306;Database=localdb;User=sa;Password=P@SS;
    networks:
      - app_network

  nginx:
    image: nginx:latest
    container_name: nginx_container
    restart: unless-stopped
    depends_on:
      - webapi
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - certbot_etc:/etc/letsencrypt # Mount Certbot SSL Certificates
      - certbot_www:/var/www/certbot # ACME Challenge folder
    networks:
      - app_network
    ports:
      - "80:80"
      - "443:443"

  certbot:
    image: certbot/certbot:latest
    container_name: certbot
    restart: unless-stopped
    volumes:
      - certbot_etc:/etc/letsencrypt
      - certbot_www:/var/www/certbot # ACME Challenge folder
    depends_on:
      - nginx
    entrypoint: >
      sh -c "certbot certonly --webroot -w /var/www/certbot -d yourdomain.com --email your@email.com --agree-tos --no-eff-email --force-renewal && certbot renew --dry-run"

networks:
  app_network:

volumes:
  mysql_data:
  certbot_etc:
  certbot_var:

```

## 11. Adjust Ngnix To Get SSL

add 

```
location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
```

```json
events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    gzip  on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://webapi:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Port $server_port;
        }
        
        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location /swagger/ {
            proxy_pass http://webapi:8080/swagger/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Port $server_port;
        }
    }
}
```

after successful Registered SSL

## 12. Last Step Adjust Ngnix To get the 443 Req

```
events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    gzip  on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

    # Redirect HTTP to HTTPS
    server {
        listen 80;
        server_name yourdomain.com;

        # ACME Challenge for Certbot (Allows HTTP validation for SSL)
        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            proxy_pass http://webapi:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        server_name yourdomain.com;

        ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

        location / {
            proxy_pass http://webapi:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location /swagger/ {
            proxy_pass http://webapi:8080/swagger/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

}

```
To start the containers, I ran:

```
docker-compose up -d

```

Now, users can access the API using:

```
http://mmydomain/swagger

```

This setup allows a fully containerized ASP.NET Web API with **MySQL and Nginx reverse proxy** accessible from external devices.

