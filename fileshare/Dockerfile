FROM python:3.11-slim

WORKDIR /app

# COPY requirements.txt .

# RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8080

CMD ["python", "-m", "http.server", "8080", "--bind", "0.0.0.0", "-d", "/usr/share/content"]
