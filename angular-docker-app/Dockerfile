FROM node:20-alpine AS build
WORKDIR /app
COPY angular-docker-app/package*.json ./
RUN npm install
COPY angular-docker-app/ ./
RUN npm run build -- --configuration=production

FROM nginx:alpine
COPY --from=build /app/dist/angular-app/browser /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
