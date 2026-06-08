FROM nginx:alpine

# Konfiguracja z endpointem /health dla health-check akcji
COPY default.conf /etc/nginx/conf.d/default.conf

# App Service for Containers domyslnie oczekuje portu 80
EXPOSE 80
