# Verdaccio

Verdaccio 搭建 npm 私有仓库

```shell
# 部署
V_PATH=/data/npm/npm-data;docker run -itd --name verdaccio \
-p 8080:4873 \
-v $V_PATH/conf:/verdaccio/conf \
-v $V_PATH/storage:/verdaccio/storage \
-v $V_PATH/plugins:/verdaccio/plugins \
verdaccio/verdaccio:5.30.2

# 重要文件：
/verdaccio/conf/config.yaml
/verdaccio/storage/htpasswd
/verdaccio/storage/data
```

TODO：
1. 账号创建
2. 本地项目打包并推到指定 npm 库
