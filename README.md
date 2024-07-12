# djangodockerforpi

# Deploying a Simple Django Web App on Raspberry Pi

## 1. Create a Simple Django Project on Your Local Machine

### Step 1: Set Up the Project Directory
1. Create a project directory:
    ```bash
    mkdir my_django_project
    cd my_django_project
    ```

2. Create a virtual environment and activate it:
    ```bash
    python3 -m venv venv
    source venv/bin/activate
    ```

3. Install Django:
    ```bash
    pip install django
    ```

4. Start a new Django project:
    ```bash
    django-admin startproject mysite .
    ```

5. Create a new Django app:
    ```bash
    python manage.py startapp myapp
    ```

6. Add the new app to your project settings in `mysite/settings.py`:
    ```python
    INSTALLED_APPS = [
        ...
        'myapp',
    ]
    ```

7. Create a simple view in `myapp/views.py`:
    ```python
    from django.http import HttpResponse

    def home(request):
        return HttpResponse("Hello, world!")
    ```

8. Map the view to a URL in `myapp/urls.py`:
    ```python
    from django.urls import path
    from . import views

    urlpatterns = [
        path('', views.home, name='home'),
    ]
    ```

9. Include the app’s URLs in the main project’s URLs in `mysite/urls.py`:
    ```python
    from django.contrib import admin
    from django.urls import include, path

    urlpatterns = [
        path('admin/', admin.site.urls),
        path('', include('myapp.urls')),
    ]
    ```

10. Run migrations and start the development server to test:
    ```bash
    python manage.py migrate
    python manage.py runserver
    ```

    Visit `http://127.0.0.1:8000/` to see your app running.

## 2. Dockerize Your Project and Run Locally

### Step 1: Create a Dockerfile
1. Create a `Dockerfile` in the project root:
    ```Dockerfile
    # Use the official Python image from the Docker Hub
    FROM python:3.9-slim

    # Set environment variables
    ENV PYTHONDONTWRITEBYTECODE 1
    ENV PYTHONUNBUFFERED 1

    # Create and set the working directory
    WORKDIR /code

    # Install dependencies
    COPY requirements.txt /code/
    RUN pip install -r requirements.txt

    # Copy project files
    COPY . /code/
    ```

2. Create a `requirements.txt` file with necessary dependencies:
    ```bash
    django
    gunicorn
    ```

3. Create a `docker-compose.yml` file:
    ```yaml
    version: '3.8'

    services:
      web:
        build: .
        command: gunicorn mysite.wsgi:application --bind 0.0.0.0:8000
        volumes:
          - .:/code
        ports:
          - "8000:8000"
    ```

4. Build and run the Docker containers:
    ```bash
    docker-compose up --build
    ```

    Visit `http://127.0.0.1:8000/` to see your containerized app running.

## 3. Deploy Your Dockerized Project on a Raspberry Pi Using SSH/SCP

### Step 1: Prepare the Raspberry Pi
1. Install Raspberry Pi OS and set up your Raspberry Pi.
2. Update and upgrade the system:
    ```bash
    sudo apt update
    sudo apt upgrade
    ```

3. Enable SSH:
    ```bash
    sudo raspi-config
    ```
    Go to `Interfacing Options` > `SSH` and enable it.

4. Install Docker and Docker Compose:
    ```bash
    curl -sSL https://get.docker.com | sh
    sudo usermod -aG docker $USER
    sudo apt-get install -y libffi-dev libssl-dev
    sudo apt-get install -y python3 python3-pip
    sudo apt-get remove python-configparser
    sudo pip3 install docker-compose
    ```

### Step 2: Transfer Project Files
1. Transfer your project files to the Raspberry Pi using `scp`:
    ```bash
    scp -r my_django_project pi@<raspberry_pi_ip>:/home/pi/
    ```

2. SSH into your Raspberry Pi:
    ```bash
    ssh pi@<raspberry_pi_ip>
    ```

3. Navigate to your project directory:
    ```bash
    cd /home/pi/my_django_project
    ```

4. Build and run your containers on the Raspberry Pi:
    ```bash
    docker-compose up --build -d
    ```

## 4. Configure Your Raspberry Pi and Webapp for Access via Web

### Step 1: Configure Your Router
1. Log in to your router’s web interface.
2. Find the port forwarding section.
3. Add a new rule to forward HTTP (port 80) and HTTPS (port 443) traffic to the static IP address of your Raspberry Pi (e.g., 192.168.1.100).

### Step 2: Set Up a Static IP for Your Raspberry Pi
1. Edit the DHCP client configuration:
    ```bash
    sudo nano /etc/dhcpcd.conf
    ```

2. Add the following lines at the end of the file (adjust to your network configuration):
    ```
    interface eth0
    static ip_address=192.168.1.100/24
    static routers=192.168.1.1
    static domain_name_servers=192.168.1.1
    ```

3. Restart the DHCP service:
    ```bash
    sudo service dhcpcd restart
    ```

### Step 3: Install and Configure Nginx
1. Install Nginx:
    ```bash
    sudo apt-get install nginx
    ```

2. Create an Nginx configuration file for your project:
    ```bash
    sudo nano /etc/nginx/sites-available/my_django_project
    ```

    Add the following configuration:
    ```nginx
    server {
        listen 80;
        server_name <your_domain_or_public_ip>;

        location / {
            proxy_pass http://127.0.0.1:8000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
    ```

3. Enable the configuration and restart Nginx:
    ```bash
    sudo ln -s /etc/nginx/sites-available/my_django_project /etc/nginx/sites-enabled
    sudo nginx -t
    sudo systemctl restart nginx
    ```

### Step 4: Secure Your Site with SSL
1. Set up SSL using Let's Encrypt:
    ```bash
    sudo apt-get install certbot python3-certbot-nginx
    sudo certbot --nginx
    ```

2. Follow the prompts to complete the SSL setup.

### Step 5: Access Your Site from the Web
1. Once everything is set up, you can access your site by navigating to your domain name or public IP address in a web browser.
