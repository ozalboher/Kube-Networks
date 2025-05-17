FROM node:14-alpine AS builder

WORKDIR /app

COPY package.json .

RUN npm install

COPY . .

RUN npm run build

FROM nginx:1.19-alpine
# Copy the react build folder(from previous stage) to nginx html folder
COPY --from=builder /app/build /usr/share/nginx/html 
# replace the default nginx.conf with our own
COPY conf/nginx.conf /etc/nginx/conf.d/default.conf
# Expose port 80 to the outside world
EXPOSE 80
# Start nginx server
CMD [ "nginx", "-g", "daemon off;" ]