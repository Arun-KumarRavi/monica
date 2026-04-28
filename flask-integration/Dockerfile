# Use lightweight Python image
FROM python:3.9-slim
 
# Set working directory
WORKDIR /usr/src/app
 
# Install system dependencies (for pandas, sklearn)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
&& rm -rf /var/lib/apt/lists/*
 
# Copy all backend files
COPY . .
 
# Install Python dependencies
RUN pip install --no-cache-dir flask flask-cors pandas scikit-learn numpy
 
# Expose Flask port
EXPOSE 5000
 
# Run the Flask app
CMD ["python", "app.py"]
