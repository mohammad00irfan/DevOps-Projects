---

## 📌 Task Overview

* Dockerize a Python application
* Install dependencies using `requirements.txt`
* Expose application on port **8087**
* Run the app using `server.py`
* Deploy container and test using `curl`

---

## 📁 Directory Structure

```text
/python_app
├── Dockerfile
└── src
    ├── requirements.txt
    └── server.py
```

---

## 🐳 Dockerfile

The Dockerfile is created under `/python_app` with the following configuration:

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY src/requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY src/ .

EXPOSE 8087

CMD ["python", "server.py"]
```

---

## 🛠️ Build Docker Image

Run the following command from the `/python_app` directory:

```bash
docker build -t nautilus/python-app .
```

Verify the image:

```bash
docker images | grep nautilus
```

---

## 🚀 Run the Container

Create and start the container with required port mapping:

```bash
docker run -d \
--name pythonapp_nautilus \
-p 8099:8087 \
nautilus/python-app
```

Verify container status:

```bash
docker ps | grep pythonapp_nautilus
```

---

## 🔍 Test the Application

Test the deployed application on **App Server 1**:

```bash
curl http://localhost:8099
```

A valid response confirms the application is running successfully.

---

## ✅ Completion Checklist

* Dockerfile created ✔️
* Image built: `nautilus/python-app` ✔️
* Container running: `pythonapp_nautilus` ✔️
* Port mapped: `8099 → 8087` ✔️
* Application tested via `curl` ✔️

---

## 📄 Notes

* Any Python base image can be used
* Port mapping allows external access via host port `8099`
* Designed for App Server–based deployment

---
