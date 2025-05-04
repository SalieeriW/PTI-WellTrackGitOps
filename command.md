❯ kubectl create secret docker-registry harbor-credentials \
  --docker-server=172.17.0.1:80 \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  --docker-email=admin@example.com \
  -n welltrack