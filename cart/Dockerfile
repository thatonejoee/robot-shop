FROM node:18.20.4-slim

EXPOSE 8080

WORKDIR /opt/server

COPY package.json /opt/server/

RUN npm install --omit=dev && npm install --omit=dev @opentelemetry/auto-instrumentations-node@0.56.1

COPY server.js /opt/server/

CMD ["node", "--require", "@opentelemetry/auto-instrumentations-node/register", "server.js"]
