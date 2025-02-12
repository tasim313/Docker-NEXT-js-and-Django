### Project Structure

Create a project folder:
```bash
my-app/
│── backend/    # Django REST API
│── frontend/   # Next.js app
│── nginx/      # Nginx configuration
│── docker-compose.yml
│── .env


```

1. Create a Dockerfile for Django (Backend)

Inside the backend/ folder, create a Dockerfile:
```bash
# Use official Python image
FROM python:3.10

# Set working directory inside the container
WORKDIR /app

# Copy the project files
COPY . .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Expose the Django default port
EXPOSE 8000

# Run migrations and start Django server
CMD ["sh", "-c", "python manage.py migrate && python manage.py runserver 0.0.0.0:8000"]

```

2. Create a Dockerfile for Next.js (Frontend)
   
Inside the frontend/ folder, create a Dockerfile:
```bash
# Use official Node.js image
FROM node:18

# Set working directory inside the container
WORKDIR /app

# Copy package.json and install dependencies
COPY package.json package-lock.json ./
RUN npm install

# Copy the rest of the frontend code
COPY . .

# Build Next.js application
RUN npm run build

# Expose Next.js default port
EXPOSE 3000

# Start the Next.js application
CMD ["npm", "run", "start"]

```

3. Create a docker-compose.yml File
   
In the my-app/ root folder, create docker-compose.yml:
```bash
version: '3.8'

services:
  nginx:
    image: nginx:latest
    container_name: nginx-proxy
    depends_on:
      - backend
      - frontend
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "80:80"
      - "443:443"
    networks:
      - my_network
    restart: always

  db:
    image: postgres:15
    container_name: postgres-db
    restart: always
    ports:
      - "5432:5432"
    env_file:
      - .env
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - my_network

  redis:
    image: redis:alpine
    container_name: redis-cache
    restart: always
    ports:
      - "6379:6379"
    networks:
      - my_network

  backend:
    build: ./backend
    container_name: django-backend
    depends_on:
      - db
      - redis
    env_file:
      - .env
    volumes:
      - ./backend:/app
    ports:
      - "8000:8000"
    networks:
      - my_network
    restart: always

  frontend:
    build: ./frontend
    container_name: nextjs-frontend
    depends_on:
      - backend
    volumes:
      - ./frontend:/app
    ports:
      - "3000:3000"
    networks:
      - my_network
    restart: always

  celery:
    build: ./backend
    container_name: celery-worker
    command: celery -A myproject worker --loglevel=info
    depends_on:
      - backend
      - redis
    env_file:
      - .env
    networks:
      - my_network
    restart: always

  celery-beat:
    build: ./backend
    container_name: celery-beat
    command: celery -A myproject beat --loglevel=info
    depends_on:
      - backend
      - redis
    env_file:
      - .env
    networks:
      - my_network
    restart: always

networks:
  my_network:
    driver: bridge

volumes:
  postgres_data:

```
4. Create nginx/nginx.conf
   
This will act as a reverse proxy for Django API and Next.js.
```bash
events {}

http {
    upstream backend {
        server backend:8000;
    }

    upstream frontend {
        server frontend:3000;
    }

    server {
        listen 80;

        location /api/ {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location / {
            proxy_pass http://frontend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}

```
5. Create a .env File

In the my-app/ root folder, create a .env file:
```bash
POSTGRES_DB=mydatabase
POSTGRES_USER=myuser
POSTGRES_PASSWORD=mypassword

```
6. Build & Run Everything 🚀

Run this command in the my-app/ directory:
```bash
docker-compose up --build
```
