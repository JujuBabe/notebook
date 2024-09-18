# Sourcegraph

代码全局搜索。gitlab中有插件，只是需要付费，可自行搭建。但是仅有10个免费的成员用户，使用时需注意权限管理问题。

```shell

docker run -d --name sourcegraph --publish 7080:7080 --publish 3370:3370 --volume $PWD/config:/etc/sourcegraph --volume $PWD/data:/var/opt/sourcegraph sourcegraph/server:4.3.1

# 增加内存、cpu资源限制
docker run -d --name sourcegraph --publish 7080:7080 --publish 3370:3370 -m 2000M --memory-swap -1 --cpus=1 --volume $PWD/config:/etc/sourcegraph -e TZ=Asia/Shanghai -v /etc/localtime:/etc/localtime:ro -v $PWD/data:/var/opt/sourcegraph sourcegraph/server:4.3.1


# 增加内存、cpu资源限制后服务容易挂，监控健康状态，不健康时实现重启
docker run -d \
--name sourcegraph \
--publish 7080:7080 \
--publish 3370:3370 \
-m 2000M \
--memory-swap -1 \
--cpus=1 \
--restart=always \
--health-cmd="curl -fs http://10.6.10.119:7080 || exit 1" \
--health-interval=30s \
--health-retries=3 \
--health-timeout=2s \
--volume $PWD/config:/etc/sourcegraph \
-e TZ=Asia/Shanghai \
-e autoheal=true \
-v /etc/localtime:/etc/localtime:ro \
-v $PWD/data:/var/opt/sourcegraph \
sourcegraph/server:4.3.1

# source 同步内容太多容易把资源拉爆，因此需要对容器占用资源进行限制，并且进行状态监控

# 查看健康检查最近的5条历史：
docker inspect --format '{{json .State.Health}}' sourcegraph

# 异常自愈
docker run -d \
    --name autoheal \
    --restart=always \
    -e AUTOHEAL_CONTAINER_LABEL=all \
    -v /var/run/docker.sock:/var/run/docker.sock \
    willfarrell/autoheal
    
# 限制cpu 使用率，防止频繁cpu告警
docker update --cpus 0.5 --cpuset-cpus 0 sourcegraph
```