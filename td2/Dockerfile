FROM node:24 AS build

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build
RUN npm prune --omit=dev


FROM node:24-alpine

WORKDIR /app

COPY --from=build --chown=node:node /app/package*.json ./
COPY --from=build --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist

EXPOSE 3000

USER node

CMD ["node", "dist/server.js"]