FROM python:3.9-slim

# Install Nginx to serve static files
RUN apt-get update && apt-get install -y nginx && rm -rf /var/lib/apt/lists/*

# Remove default nginx static assets
RUN rm -rf /usr/share/nginx/html/*

# Copy our static website
COPY index.html /usr/share/nginx/html/index.html

# Expose port 80 to the internal Docker network
EXPOSE 80

# Start Nginx
CMD ["nginx", "-g", "daemon off;"]