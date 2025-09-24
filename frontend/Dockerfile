# frontend/Dockerfile

# Use an official Nginx runtime as a parent image
FROM nginx:alpine

# Copy the static content (HTML, JS) to the Nginx server directory
COPY . /usr/share/nginx/html

# Copy the custom Nginx configuration file
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose port 80
EXPOSE 80