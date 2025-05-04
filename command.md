
# 在镜像被设置为公开后，尝试拉取
docker pull ghcr.io/hongda-zhu/welltrack-frontend:latest

# 加载到 Kind 集群
kind load docker-image ghcr.io/hongda-zhu/welltrack-frontend:latest --name welltrack-local  