# deploy-xray-sni
![Ansible](https://img.shields.io/badge/automation-Ansible-blue?style=flat-square)
![Xray](https://img.shields.io/badge/xray-supported-purple?style=flat-square)

С помощью данного гайда вы сможете развернуть собственный SNI с сертификатом DNS-01, который можно использовать, как заглушку для VLESS-REALITY.

Ниже описано два метода запуска:
1. [В ручном режиме](#запуск-в-ручном-режиме)
2. [В автоматическом режиме (Ansible)](#запуск-в-автоматическом-режиме-ansible)
> [!WARNING] 
> Для работы SNI нужен домен, зарегистрированный в Cloudflare!
---
## Запуск в ручном режиме 
1. Сначала [получаем API-токен Cloudflare](#получение-токена-cloudflare)
2. Копируем репозиторий с SNI заглушкой:
```sh
git clone https://github.com/locklance/xray-sni.git
```
3. Создаем и заполняем `.env` файл, подставляя API-токен
```sh
cd xray-sni
vim .env
----- Заполняем .env: -----
  SNI_DOMAIN="sni.example.com" # SNI dest address
  SNI_PORT="9443" # SNI dest port
  CF_API_TOKEN="YOUR_CF_API_TOKEN" # Cloudflare API token
---------------------------
```
4. Запускаем SNI
```sh
sudo docker compose up -d && docker compose logs -ft
```

## Запуск в автоматическом режиме (Ansible)
Запуск в автоматическом режиме может показаться сложным, но на самом деле он очень простой и занимает куда меньше времени если у вас два и более серверов!
### Подготовка
1. Добавляем ssh-ключи на своем ПК через ssh-agent
Для безопасности, чтобы не добавлять путь до каждого ключа в inventory файле, добавляем ssh ключ в систему через ssh-add:
```sh
ssh-add .ssh/your_key
```
2. Создаем venv
```sh
python -m venv venv
source venv/bin/activate
```
3. Устанавливаем Ansible
```sh
pip install ansible
```
4. Устанавливаем роль Ansible
Устанавливаем роль для автоматической загрузки docker в систему:
```sh
ansible-galaxy install geerlingguy.docker
```
5. Настраиваем inventory файл
Самый важный пункт, в котором мы должны заполнить файл `ansible/inventory.yml` данными для ssh соединения с серверами.
Пример:
```yml
vpnnodes:
  hosts:
    DE_NODE_1:
      ansible_host: 127.0.0.1 <---- SERVER IP ADDRESS
      ansible_port: 22 <----SSH PORT
      ansible_user: DE_NODE_1 <---- SSH USER
    DE_NODE_2:
      ansible_host: 127.0.0.2
      ansible_port: 23
      ansible_user: DE_NODE_2
germany:
  hosts:
    DE_NODE_1:
    DE_NODE_2:
  vars:
    domain: de.example.com <---- DOMAIN
```
### Проверка
Проверка правильности файла Ansible inventory:
```sh
ansible-inventory -i ansible/inventory.yml --list
```
Пингуем все сервера из inventory файла:
`ansible vpnnodes -m ping -i ansible/inventory.yml`
вместо `vpnnodes`, можем использовать любую другую группу или отдельную страну (например `finland`, `FL_NODE_1`). 

### Запуск
1. Создаем секрет `secrets.yml` с API-ключом Cloudflare `CF_API_TOKEN`:
```sh
└── roles
    └── sni
        ├── files
        │   └── secrets.yml <---- PUT CF_API_TOKEN HERE
        ├── tasks
        │   └── main.yml
        └── templates
            └── env.j2
```
Пример **secrets.yml**:
```sh
# secrets.yml example
CF_API_TOKEN: your_api_token
```

2. Запускаем Ansible скрипт:
```sh
ansible-playbook ansible/playbooks/deploy_sni.yml -i ansible/inventory.yml -l finland
```
> \* Используйте опцию `--ask-become-pass` для исполнения sudo команд на сервере если не сработало без неё

> \*\* Чтобы использовать собственный sni, в файле `deploy_sni.yml` поменяйте переменную `repo_url` на url репозитория вашего собственного SNI:
> ```yml
> vars:
>     docker_install_compose: true
>     docker_group: docker
>     repo_url: "https://github.com/locklance/xray-sni" <---- SNI URL
>     dest_subdir: "xray-sni"
>     ...
> ```

## Получение токена Cloudflare
### Регистрация домена в Cloudflare
В первую очередь, нам необходимо пройти регистрацию домена. Создаем какой-нибудь правдоподобный домен, чтобы он вызывал меньше подозрений.
![Регистрация домена](/docs/media/create-domain.png)

### Получаем API-токен
1. Переходиим в личный профиль
![Заходим в профиль](/docs/media/cf-profile.png)
2. Находим раздел API Tokens
![Заходим в раздел API Tokens](/docs/media/get-api-key-2.png)
3. Создаем новый токен
![Создаем новый токен](/docs/media/get-api-key-3.png)
4. Добавляем правила токену
![Добавляем правила токена](/docs/media/get-api-key-4.png)