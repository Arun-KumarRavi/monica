FROM python:3.9-slim
WORKDIR /usr/src/app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# --- THE FIX: Copy ONLY dependency-related files first ---
# (Since you don't have a requirements.txt, we can use app.py as a proxy 
# or create a requirements.txt to make this even faster)
COPY New_Bucket.py . 
RUN pip install --no-cache-dir flask flask-cors pandas scikit-learn numpy

# --- Now copy the rest of the code ---
COPY . .

EXPOSE 5000
CMD ["python", "app.py"]
