# Dockerfile

基本基于Gin的微服务Dockerfile

```dockerfile
FROM golang:1.17 AS build
ENV GO_PROXY=https://goproxy.cn
WORKDIR /root/project
ENV CGO_ENABLED=0
RUN go mod tidy
RUN go build -tags=release -a -installsuffix cgo  -o ./main


FROM alpine:3.15.0 AS runtime
COPY --from=build /root/project/main .
ENV GIN_MODE=release
EXPOSE 8080/tcp
ENTRYPOINT ["./main"]
```
