# docker 命令

## 启动容器
```bash
docker start 容器名称
```

## 查看启动的容器
```bash
docker ps
```

## 进入容器的 Shell
```bash
docker exec -it 容器名称 bash
```

## 进入 PostgreSQL 命令行，不需要先进入 Shell，直接执行
```bash
docker exec -it xinghao-db-postgres psql -U xinghao_db -d test_db
```


