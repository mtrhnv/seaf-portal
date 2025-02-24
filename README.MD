# Развертывание SEAF ArchPortal


В данном сценарии описывается развертывание портала SEAF вместе с Keycloak и Gitlab. Сценарий можно адаптировать под развертывание в промышленных средах.
## Все на одной машине
На одной машине будут развернуты контейнеры SEAF Archtool, PlantUML, Keycloak (postgres), Gitlab и Reverse-Proxy на NGINX.

| Контейнер     | Порты и протоколы            |
|---------------|------------------------------|
| seaf-backend  | tcp/8080                     |
| plantuml      | tcp/8079                     |   
| keycloak      | tcp/8443, tcp/9000, tcp/8081 |
| gitlab        | tcp/8929, tcp/8083, tcp/2424 |
| postgres      | tcp/5432                     |
| reverse-proxy | tcp/80, tcp/443              |

Кроме этого набора также необходимо развернуть контейнеры airflow https://github.com/xantia88/mle-airflow

### Подготовка к развертыванию
1. Создайте FQDN для сервисов ниже и зарегистрируйте на вашем DNS сервере

| Контейнер    | FQDN (пример)        |
|--------------|----------------------|
| seaf-backend | archportal.local     |
| keycloak     | aaa.archportal.local |   
| gitlab       | git.archportal.local |

2. Склонируйте данный репозиторий и инциализируйте сабмодули
```shell
git clone https://github.com/mtrhnv/seaf-portal.git

cd seaf-portal

git submodule init && git submodule update --recursive
```

3. Создайте и подпишите на вашем CA сертификаты для каждого FQDN. 

Для тестового развертывания мы создадим самоподписанный wildcard сертификат для домена **archportal.local**

**В консоли перейдите в корень клонированного репозитория!**
```shell
# Зададим наш домен
DOMAIN=archportal.local
# Создадим и перейдем в папку с сертификатами
mkdir certs
cd certs
# Создадим конфигурационный файл
cat > openssl.conf << EOF
[CA_default]
copy_extensions = copy

[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = req_distinguished_name
x509_extensions = v3_ca

[req_distinguished_name]
C = RU
ST = Moscow
L = Moscow
O = Lazy Arch Inc
OU = IT
emailAddress = it@nowhere.earth
CN = $DOMAIN

[v3_ca]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
subjectKeyIdentifier = hash
subjectAltName = @alternate_names

[alternate_names]
DNS.1 = $DOMAIN
DNS.2 = *.$DOMAIN
EOF
# Сгенерируем сертификаты
openssl req -x509 -newkey rsa:2048 -sha256 -utf8 -days 365 -nodes \
    -config ./openssl.conf \
    -keyout ./server.key \
    -out ./server.crt

cd ..
cp certs/server.crt certs/server.key reverse-proxy/certs
```
Теперь у нас есть сертификаты и можно поднимать наши сервисы.

4. Запускаем Reverse-Proxy
``` shell
cd reverse-proxy
docker-compose up -d
cd ..
```
5. Запускаем Keycloak или настраиваем свой OpenID Identity Provider
``` shell
cd keycloak
docker-compose up -d
cd ..
```
Далее необходимо настроить Keycloak для работы в нашей инсталляции по [инструкции](keycloak/README.MD)

Если вы используете ADFS, то настройка описана здесь: [инструкции](keycloak/ADFS.MD)

6. Запускаем Gitlab
``` shell
cd keycloak
docker-compose up -d
cd ..
```
Далее необходимо настроить Gitlab для работы в нашей инсталляции по [инструкции](gitlab/README.MD)

7. Запускаем SEAF ArchPortal
Перед запуском убедитесь в корректности параметров в **.env**
``` shell
docker-compose up -d
cd ..
```
Подробную **инструкцию по реализации ролевой модели** вы можете найти [здесь](ROLEMODEL.MD)


## Развертывание на разных машинах
Для развертывания на нескольких виртуальных машинах вам потребуется:
1. Cгенерировать сертификаты как в п.2 инструкции по развертыванию на одной машине
2. Скопировать весь пакет вместе с сгенерированными сертификатами на все машины
3. Запустить на каждой машине Reverse-Proxy и соответствующий сервис повторяя шаги [4,5], [4,6], [4,7]
4. Необходимо обеспечить сетевую доступность между виртуальными машинами и клиентами в соответствии с таблицей 

| Src                      | Dst                            | Protocol/port   |
|--------------------------|--------------------------------|-----------------|
| Пользовательский сегмент | seaf-backend, keycloak, gitlab | tcp/80, tcp/443 |
| seaf-backend             | keycloak, gitlab               | tcp/443         |   
