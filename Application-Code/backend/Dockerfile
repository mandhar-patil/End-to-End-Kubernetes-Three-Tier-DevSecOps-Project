FROM node:18-alpine

WORKDIR /usr/src/app

# Update alpine packages
RUN apk update && apk upgrade

COPY package*.json ./

RUN npm install

COPY . .

CMD ["node", "index.js"]
