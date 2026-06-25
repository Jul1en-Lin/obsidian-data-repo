# 环境配置
在做脚手架之前，需要对服务器进行一些中间件的配置，文件位于 `deploy/dev/app` 中，包括 docker 的 mysql、redis、rabbitmq、nacos 等，上传该文件到服务器中，再进行 docker 容器初始化。
## Docker-compose
一系列的 docker 配置文件都保存在 `docker-compose-mid.yml` 中，在服务器通过命令一键拉起。yml 文件中**挂载**了一系列的数据文件
### 命令
启动
`docker compose -p framework-java -f docker-compose-mid.yml up -d`
停止
`docker compose -p framework-java -f docker-compose-mid.yml stop
