FROM node:20-alpine3.18

WORKDIR /app

ENV NODE_ENV=development

COPY package.json .
COPY package-lock.json .

RUN npm ci

COPY . .

CMD [ "npm", "run", "dev" ]