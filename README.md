# Foodgram React





## Описание

 «Продуктовый помощник»: сайт, на котором пользователи будут публиковать рецепты, добавлять чужие рецепты в избранное и подписываться на публикации других авторов. Сервис «Список покупок» позволит пользователям создавать список продуктов, которые нужно купить для приготовления выбранных блюд. 

[![workflow food](https://github.com/SerkaFox/foodgram-project-react/actions/workflows/main.yml/badge.svg)
-


# Проект **foodgram-project-react** 

#### ****
[![gunicorn](https://img.shields.io/badge/-gunicorn-464646?style=flat-square&logo=gunicorn)](https://gunicorn.org/)
[![PostgreSQL](https://img.shields.io/badge/-PostgreSQL-464646?style=flat-square&logo=PostgreSQL)](https://www.postgresql.org/)
[![Nginx](https://img.shields.io/badge/-NGINX-464646?style=flat-square&logo=NGINX)](https://nginx.org/ru/)


## Зависимости
```
- Перечислены в файле backend/requirements.txt
```
Локальная проверка 'http://localhost:8000' 

Проверка на сервере
```
- ставим докер и композ
sudo apt update
sudo apt install docker.io
sudo apt install docker-compose
sudo systemctl start docker

- зайти в свой проект на компе, проверить docker-compose.yml
```
В директории infra/, в файле nginx.conf измените адрес(ip/домен), необходимо указать адрес вашего сервера.

Запустите docker compose
```
docker-compose up -d --build
```

Примените миграции
```
docker-compose exec backend python manage.py migrate
(при необходимости наполнения БД можно загрузить базу с помощью:
docker-compose exec backend python manage.py loading_ingredients)
что такое loading_ingredients? - это не ручками забить 2000 ингридиетов,
а испытать Дзен от лени и удобства 
```

ssh serka@158.160.46.202

```
- заходим на сервер

- ставим докер и композ
sudo apt update
sudo apt install docker.io
sudo apt install docker-compose
sudo systemctl start docker

```

зайти в свой проект на компе, проверить докерфайл,композ и воркфлоу
```
- скопируйте файлы docker-compose.yaml и nginx.conf на сервер
cd ~
touch docker-compose.yml
nano docker-compose.yml
копировать содержимое файла на локальном компе и вставить в файл на сервере


touch nginx.conf
nano nginx.conf
копировать содержимое файла на локальном компе и вставить в файл на сервере
---------------------------------------------------------------------

server {
    listen 80;
    server_name 158.160.46.202;
    server_tokens off;

    location /api/docs/ {
        root /usr/share/nginx/html;
        try_files $uri $uri/redoc.html;
    }

    location /back_static/ {
        root /var/html/;
    }

    location /back_media/ {
        root /var/html/;
    }

    location /api/ {
        proxy_pass http://backend:8000;
        proxy_set_header        Host      $host;
        proxy_set_header        X-Real-IP $remote_addr;
    }

    location /admin/ {
        proxy_pass http://backend:8000/admin/;
        proxy_set_header        Host      $host;
        proxy_set_header        X-Real-IP $remote_addr;
    }

    location / {
        root /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri /index.html;
        proxy_set_header        Host $host;
        proxy_set_header        X-Real-IP $remote_addr;
        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header        X-Forwarded-Proto $scheme;
      }
      error_page   500 502 503 504  /50x.html;
      location = /50x.html {
        root   /var/html/frontend/;
      }

}

================================================================================
version: '3.3'
services:

  frontend:
    image: serkafox/foodgram_frontend:v1.0
    volumes:
      - ../frontend/:/app/result_build/

  db:
    image: postgres:13.0-alpine
    volumes:
      - postgres:/var/lib/postgres/data/
    env_file:
      - ./.env

  nginx:
    image: nginx:1.19.3
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
      - ../frontend/build:/usr/share/nginx/html/
      - docs:/usr/share/nginx/html/api/docs/
      - static_value:/var/html/back_static/
      - media_value:/var/html/back_media/
    depends_on:
      - backend

  backend:
    image: serkafox/foodgram_backend:v1.0
    restart: always
    volumes:
      -  static_value:/app/back_static/
      -  media_value:/app/back_media/
      -  docs:/app/api/docs/
    depends_on:
      - db
    env_file:
      - ./.env

volumes:
  static_value:
  media_value:
  postgres:
  docs:
=====================================================================================================

name: foodgram_workflow


on: [push]

jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: 3.10.6

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r backend/requirements.txt 
  build_and_push_to_docker_hub:
      name: Push Docker image to Docker Hub
      runs-on: ubuntu-latest
      needs: tests
      steps:
        - name: Check out the repo
          uses: actions/checkout@v2
        - name: Set up Docker Buildx
          uses: docker/setup-buildx-action@v1
        - name: Login to Docker
          uses: docker/login-action@v1
          with:
            username: ${{ secrets.DOCKER_USERNAME }}
            password: ${{ secrets.DOCKER_PASSWORD }}
        - name: Push to Docker Hub
          uses: docker/build-push-action@v2
          with:
            context: ./backend/
            push: true
            tags: ${{ secrets.DOCKER_USERNAME }}/foodgram_backend:v1.0

  deploy:
    runs-on: ubuntu-latest
    needs: build_and_push_to_docker_hub
    steps:
      - name: executing remote ssh commands to deploy
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            sudo docker-compose stop
            sudo docker-compose rm backend
            sudo rm .env
            touch .env
            echo DB_ENGINE=${{ secrets.DB_ENGINE }} >> .env
            echo DB_NAME=${{ secrets.DB_NAME }} >> .env
            echo POSTGRES_USER=${{ secrets.POSTGRES_USER }} >> .env
            echo POSTGRES_PASSWORD=${{ secrets.POSTGRES_PASSWORD }} >> .env
            echo DB_HOST=${{ secrets.DB_HOST }} >> .env
            echo DB_PORT=${{ secrets.DB_PORT }} >> .env

            echo SECRET_KEY=${{ secrets.SECRET_KEY }} >> .env
            echo DEBUG=${{ secrets.DEBUG }} >> .env
            echo ALLOWED_HOSTS=${{ secrets.ALLOWED_HOSTS }} >> .env
            sudo docker-compose up -d

  send_message_deploy:
    runs-on: ubuntu-latest
    needs: deploy
    steps:
      - name: send message
        uses: appleboy/telegram-action@master
        with:
          to: ${{ secrets.TELEGRAM_TO }}
          token: ${{ secrets.TELEGRAM_TOKEN }}
          message: ${{ github.workflow }} Ну что,Бразер) интересно,что дальше ? ) 
```
### **Admin:**
```
http://158.160.46.202/admin/
```

Пробуем 
```
http://158.160.46.202/signin
```
### **Автор**
Svitkin Sergey
