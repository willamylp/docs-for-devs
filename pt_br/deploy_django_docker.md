# Documentação de Deploy - Django, PostgreSQL e Nginx no Docker

Este documento descreve o processo de deploy de uma aplicação **Django** utilizando **Docker**, **Docker Compose** e **Nginx** como servidor web. A aplicação será servida por **Gunicorn**, um servidor **WSGI** para **Python**, e o banco de dados utilizado é o **PostgreSQL**.

<br>

## Estrutura do Projeto

A estrutura do projeto é organizada da seguinte forma:

```plaintext
project-root/
    ├── app/ (Diretório da aplicação Django)
        ├── settings.py
        ├── wsgi.py
        ├── ...
    ├── other_apps/ (Outros diretórios de Apps do projeto)
    ├── static/
    ├── media/
    ├── wait-for-it.sh  
    ├── Dockerfile  
    ├── docker-compose.yml  
    ├── nginx.conf  
    ├── requirements.txt
    ├── manage.py
    └── .env
```

### Arquivos importantes  

- **Dockerfile**: Define a imagem Docker para a aplicação Django.  
- **docker-compose.yml**: Define os serviços (Django, PostgreSQL, e Nginx) que serão executados em containers Docker.  
- **nginx.conf**: Configuração do servidor Nginx que faz proxy reverso para o Gunicorn.  
- **requirements.txt**: Lista todas as dependências Python da aplicação.
- **.env**: Arquivo de variáveis de ambiente para configuração dos serviços.
- **wait-for-it.sh**: Script utilizado para garantir que o banco de dados PostgreSQL esteja pronto antes de rodar as migrações e iniciar o servidor.

> **Obs.:** O arquivo `wait-for-it.sh` está disponível em:
[https://raw.githubusercontent.com/vishnubob/wait-for-it/master/wait-for-it.sh](https://raw.githubusercontent.com/vishnubob/wait-for-it/master/wait-for-it.sh).

<br>

## Ajustes no `settings.py` Recomendados para o Projeto

#### 1. Ajustar o apontamento do Banco de Dados para as variáveis de ambiente

> ***Obs.:** A biblioteca `python-decouple` é recomendada para facilitar a leitura das variáveis de ambiente.* \
> *Documentação: [https://pypi.org/project/python-decouple/](https://pypi.org/project/python-decouple/)*

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME'),
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': config('DB_HOST'),
        'PORT': config('DB_PORT'),
    }
}
```

#### 2. Estrutura recomendada para os diretórios **static** e **media**

```python
# Static files (CSS, JavaScript, Images, ...)
STATIC_URL = '/static/'
STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static'), ]
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

# Media files (Images, Files)
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
MEDIA_URL = '/media/'
```

<br>

---

<br>

# Arquivos de Configuração

## 1. Dockerfile  

O `Dockerfile` define a imagem Docker para a aplicação Django. Ele instala as dependências necessárias e copia o código da aplicação para dentro do container.

```dockerfile
# Imagem base do Python 3.12 (slim)
FROM python:3.12-slim

# Define o diretório de trabalho
WORKDIR /app

# Instala as dependências do sistema operacional 
RUN apt-get update && apt-get install -y curl

# Copia o script wait-for-it.sh para o container
COPY wait-for-it.sh /app/

# Atribui permissão de execução para o wait-for-it.sh
RUN chmod +x /app/wait-for-it.sh

# Copia o arquivo requirements.txt com as dependências do projeto
COPY requirements.txt /app/

# Instala as dependências Python do projeto
RUN pip install --no-cache-dir -r requirements.txt

# Copia o código da aplicação para o container
COPY . /app/

# Expõe a porta 8000
EXPOSE 8000
```

### Explicação  

- **python:3.12-slim**: Utiliza a imagem base do Python 3.12 (slim) para manter o container leve.

> ***Obs.:** Ajuste para a versão do Python do seu projeto.*

- **WORKDIR /app**: Define o diretório de trabalho como `/app`.
- **EXPOSE 8000**: Expondo a porta 8000 para o Django servir a aplicação.

<br>

## 2. docker-compose.yml

O `docker-compose.yml` define os serviços que serão executados em containers Docker. Ele inclui o serviço de banco de dados PostgreSQL, o serviço da aplicação Django, e o servidor Nginx.

```yaml
services:  
  db:  
    image: postgres:latest  
    container_name: postgres_db  
    volumes:  
      - postgres_data:/var/lib/postgresql/data  
    env_file:  
      - .env  
    environment:  
      POSTGRES_DB: ${DB_NAME}  
      POSTGRES_USER: ${DB_USER}  
      POSTGRES_PASSWORD: ${DB_PASSWORD}  
    ports:  
      - "5432:5432"  
    networks:  
      - app_network  
  
  web:  
    build: .  
    container_name: django_app  
    command: >  
      sh -c "./wait-for-it.sh db:5432 -- python manage.py makemigrations &&  
             python manage.py migrate &&  
             python manage.py collectstatic --noinput &&  
             gunicorn app.wsgi:application --bind 0.0.0.0:8000"  
    volumes:  
      - static_volume:/app/staticfiles  
      - media_volume:/app/media  
    env_file:  
      - .env  
    depends_on:  
      - db  
    networks:  
      - app_network  
  
  nginx:  
    image: nginx:latest  
    container_name: nginx_server  
    volumes:  
      - ./nginx.conf:/etc/nginx/nginx.conf  
      - static_volume:/app/staticfiles  
      - media_volume:/app/media  
    ports:  
      - "80:80"  
    depends_on:  
      - web  
    networks:  
      - app_network  
  
networks:  
  app_network:  
    driver: bridge  
  
volumes:  
  postgres_data:  
  static_volume:  
  media_volume:
```

### Explicação  

- **db**: Serviço do PostgreSQL. Ele utiliza o volume `postgres_data` para persistir dados e as variáveis de ambiente no arquivo `.env`.  
- **web**: Serviço da aplicação Django, que constrói a imagem a partir do `Dockerfile` e executa comandos como migrações, coleta de estáticos e inicialização do servidor Gunicorn.  

> ***Obs.:** É imprescindível que a lib **Gunicorn** esteja presente no arquivo `requirements.txt`*

- **nginx**: Serviço do servidor Nginx, configurado para servir os arquivos estáticos e fazer proxy reverso para o Gunicorn.

<br>

## 3. nginx.conf  
  
O arquivo `nginx.conf` configura o servidor Nginx, que fará o proxy reverso para o Gunicorn e servirá os arquivos estáticos e de mídia.

```nginx
worker_processes auto;  

events {  
    worker_connections 1024;  
}  
  
http {  
    sendfile on;  
    tcp_nopush on;  
    tcp_nodelay on;  
    keepalive_timeout 65;  
    types_hash_max_size 2048;  
    include /etc/nginx/mime.types;  
    default_type application/octet-stream;  
    upstream django {  
        server web:8000;  
    }  
  
    server {  
        listen 80;  
        server_name localhost;  
  
        location /static/ {  
            alias /app/staticfiles/;  
        }  
  
        location /media/ {  
            alias /app/media/;  
        }  
  
        location / {  
            proxy_pass http://django;  
            proxy_set_header Host $host;  
            proxy_set_header X-Real-IP $remote_addr;  
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  
            proxy_set_header X-Forwarded-Proto $scheme;  
        }  
    }  
}
```

### Explicação

- **upstream django**: Define o servidor Gunicorn (`web:8000`) como o destino do proxy.  
- **location /static/**: Serve os arquivos estáticos do Django.  
- **location /media/**: Serve os arquivos de mídia do Django.  
- **proxy_pass**: Redireciona as requisições para o servidor Gunicorn.  
- **worker_connections**: Este parâmetro define o número máximo de conexões que um único worker (processo) do Nginx pode manter simultaneamente. No exemplo, ele está configurado para `1024`, o que significa que cada worker pode lidar com até 1024 conexões simultâneas. Aumentar esse valor pode ser útil se você espera um grande volume de tráfego na aplicação. No entanto, valores muito altos podem levar a uma maior utilização de memória, portanto, é importante encontrar um equilíbrio com o número de workers e a carga do servidor.

<br>

## 4. Variáveis de Ambiente

O arquivo `.env` deve conter as variáveis de configuração necessárias para os serviços, como informações do banco de dados.

Exemplo de `.env`:

```env
DB_NAME=name  
DB_USER=postgres  
DB_PASSWORD=admin@123  
DB_HOST=db  
DB_PORT=5432
```

#### ⚠️ **Observação Importante:**

Como está sendo usando a configuração `networks` no `docker-compose.yml`, o *host* do Banco de Dados no arquivo `.env` precisa ser o nome do serviço do PostgreSQL, que no caso é `db`, e não `localhost` ou `127.0.0.1`.

<br>

---

<br>

## Como Rodar o Projeto

### 1. Construir as imagens e iniciar os containers  

#### Execute os comandos

```bash
docker-compose build
```

```bash
docker-compose up -d
```

### 2. Criar o superusuário Django  
  
Após iniciar os containers, crie o *superuser* Django para acessar o painel administrativo:  

```bash
docker-compose exec web python manage.py createsuperuser
```

### 3. Acessar a aplicação

A aplicação estará disponível no `http://localhost`, onde o Nginx estará servindo a aplicação Django através do Gunicorn.

<br>

---

<br>

## Situações Adversas

### 1. Precisar remover todos os Volumes e recriar

Para remover todos os volumes e recriar tudo novamente, execute o seguinte comando:

```bash
docker-compose down -v
```

Se necessário, execute o comando abaixo para limpar todo o Docker completamente:

```bash
docker system prune -a
```

Em seguida, execute novamente os comandos da seção **1. Construir as imagens e iniciar os containers**

<br>

### 2. Precisar apenas atualizar o conteúdo do container da aplicação Django

Essa ação é útil quando há apenas a modificação de um bloco de código do projeto, ou implementação de uma nova funcionalidade. Para isso, execute os seguintes comandos:

```bash
docker-compose up -d --build web
```

Isso irá subir o novo código fonte para o container da aplicação, que no exemplo chama-se `web`.
