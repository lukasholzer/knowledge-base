
# Distroless images

Distroless images only contain the app with runtime dependencies - no shells or other programms from standard Linux distributions.

## Creating a Nodejs distroless image

Example can be found on the [Github Distroless]

```Docker
FROM node:10.17.0 AS build-env
ADD . /app
WORKDIR /app

RUN npm ci --only=production

FROM gcr.io/distroless/nodejs
COPY --from=build-env /app /app
WORKDIR /app
CMD ["hello.js"]
```

[Github Distroless](https://github.com/GoogleContainerTools/distroless/tree/master/examples/nodejs)