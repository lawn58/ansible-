# ansible-

Домашнее задание
Подготовить стенд на Vagrant как минимум с одним сервером. На этом 
сервере используя Ansible необходимо развернуть nginx со следующими 
условиями:
- необходимо использовать модуль yum/apt
- конфигурационные файлы должны быть взяты из шаблона 
jinja2
 с 
переменными
- после установки nginx должен быть в режиме enabled в systemd
- должен быть использован notify для старта nginx после установки
- сайт должен слушать на нестандартном порту - 
8080
, для этого использовать 
переменные в Ansible


Домашнее задание считается принятым, если:
- предоставлен 
Vagrantfile
 и готовый 
playbook/роль

- после запуска стенда nginx доступен на порту 
8080
- при написании playbook/роли соблюдены перечисленные в задании условия
Установка Ansible


●
Версия Ansible  =>2.4 требует для своей работы Python 2.6 или выше
●
Убедитесь что у Вас установлена нужная версия:
python -V
●
Далее произведите установку для Вашей ОС по 
инструкции
 и убедитесь что 
Ansible установлен корректно:
ansible --version


Настройка Ansible
●
Для управления хостами Ansible использует SSH соединение. Поэтому 
перед стартом необходимо убедиться что у Вас есть доступ до 
управляемых хостов.
●
Также на управляемых хостах должен быть установлен Python 2.X


Подготовка окружения
●
Создайте каталог 
Ansible
 и положите в него Vagrantfile
●
Поднимите управляемый хост командой 
vagrant up
 и убедитесь что все 
прошло успешно и есть доступ по ssh
vagrant ssh
●
Для подключения к хосту 
nginx
 нам необходимо будет передать множество 
параметров - это особенность Vagrant. Узнать эти параметры можно с 
помощью команды 
vagrant ssh-config
. Вот основные необходимые нам:
vagrant ssh-config
Host nginx
     имя хоста
HostName 127.0.0.1 
     IP адрес 
User vagrant
        имя пользователя под которым подключаемся
Port 2222
  порт, который проброшен на 127.0.0.1
IdentityFile .vagrant/machines/nginx/virtualbox/private_key
путь до приватного ключа
Ansible
●
Используя эти параметры создадим свой первый 
inventory
 файл. 
Выглядеть он будет 
так
:
[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_user=vagrant 
ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key

Это все одна строка




Ansible
●
Теперь собственно приступим к выполнению домашнего задания и 
написания Playbook-а для установки 
NGINX
. Будем писать его постепенно, 
шаг за шагом. И в итоге трансформируем его в роль.
●
За основу возьмем уже созданный нами файл epel.yml (я его переименую в 
nginx.yml). И первым делом добавим в него установку пакета NGINX. 
Секция будет выглядеть так:
- name: Install nginx package from epel repo
     yum:
       name: nginx
       state: latest
     tags:
       - nginx-package
   Как видите добавлены  tags
       - packages
●

●
Обратите внимание - добавили 
Tags
. Теперь можно вывести в консоль 
список тегов и выполнить, например, только установку NGINX. В нашем 
случае так, например, можно осуществлять его обновление. 
●
Выведем в консоль все теги:

ansible-playbook nginx.yml --list-tags
playbook: epel.yml
  play #1 (nginx): NGINX | Install and configure NGINX  TAGS: []
TASK TAGS: [epel-package, nginx-package, packages]
●
Запустим только установку NGINX:

ansible-playbook nginx.yml -t nginx-package

●
Далее добавим шаблон для конфига NGINX и модуль, который будет 
копировать этот шаблон на хост:
- name: NGINX | Create NGINX config file from template
     template:
       src: templates/nginx.conf.j2
       dest: /tmp/nginx.conf
     tags:
       - nginx-configuration
●
Сразу же пропишем в Playbook необходимую нам переменную. Нам нужно 
чтобы NGINX слушал на порту 8080:
- name: NGINX | Install and configure NGINX
 hosts: nginx
 become: true
 vars:
   nginx_listen_port: 8080
В итоге на данном этапе Playbook будет выглядеть так
Добавлена только секция 
vars

●
Сам шаблон будет выглядеть так:
events {
   worker_connections 1024;
}
http {
   server {
       listen       {{ nginx_listen_port }} default_server;
       server_name  default_server;
       root         /usr/share/nginx/html;
       location / {
       }
   }
}


●
Теперь создадим 
handler
 и добавим 
notify
 к копированию шаблона. Теперь 
каждый раз когда конфиг будет изменяться - сервис перезагрузиться. 
Секция с handlers будет выглядеть следующим образом:
handlers:
   - name: restart nginx
     systemd:
       name: nginx
       state: restarted
       enabled: yes
   - name: reload nginx
     systemd:
       name: nginx
       state: reloaded
Так же создадим handler для 
рестарта и включения сервиса 
при загрузке
Перечитываем конфиг
●
Notify будут выглядеть так: 
- name: NGINX | Install NGINX package from EPEL Repo
     yum:
       name: nginx
       state: latest
     notify:
       - restart nginx
     tags:
       - nginx-package
       - packages
   - name: NGINX | Create NGINX config file from template
     template:
       src: templates/nginx.conf.j2
       dest: /etc/nginx/nginx.conf
     notify:
       - reload nginx
     tags:
       - nginx-configuration
●
Результирующий файл 
nginx.yml
. Теперь можно его запустить

ansible-playbook playbooks/nginx.yml

●
Теперь можно перейти в браузере по адресу 
http://192.168.11.150:8080
 и 
убедиться, что сайт доступен.
●
Или из консоли выполнить команду:
curl http://192.168.11.150:8080
