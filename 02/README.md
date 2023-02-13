# ДЗ 02. Установка и настройка PostgteSQL в контейнере Docker

## GCP
Чтобы себе на Windows Google Cloud CLI не ставить, будем использовать админскую машинку в облаке google

- создаем у себя ключик ```ssh-keygen -t ed25519```
- вносим его в https://console.cloud.google.com/compute/metadata?tab=sshkeys и правим имя пользователя на будущее  
- создаем в GCP машинку ```vm-admin```
    - у админской виртуалки включить "Allow full access to all Cloud APIs" 

или в ```Cloud Shell```
```
# создаем машинку vm-admin
gcloud compute instances create vm-admin --zone=europe-north1-a --machine-type=e2-small --service-account=724511005767-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/cloud-platform
```

Дальше все действия с админской машинки - подключаемся ```ssh ae@x.x.x.x``` (ip-шник внешний посмотреть в Google Cloud Console)
```
# делаем машинку для игрищ с докером 
gcloud compute instances create vm-docker --zone=europe-north1-a --machine-type=e2-small

# подключаемся через google CLI, удобно по именам и не разводим ssh ключи 
gcloud compute ssh vm-docker
```
