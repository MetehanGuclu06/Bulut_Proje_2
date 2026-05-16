FROM python:3.11-slim

WORKDIR /app

# Bağımlılıkları kopyala ve yükle
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Uygulama kodunu kopyala
COPY . .

# Port aç
EXPOSE 5000

# Gunicorn ile çalıştır (production)
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "--chdir", "/app/app", "app:app"]
