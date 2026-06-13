# # Лаб. работа №4. Ansible roles

## Подготовка
1. создаем инвентарь в `inventory.ini`, указываем один хост SLAVE 10.0.2.16:
```ini
[webservers]
10.0.2.16
```

2. создаем файл сценария `playbook.yml`, добавляем:
- хосты, на которых выполняем сценарий - группа `webservers`
- под каким пользователем логинимся на целевые хосты - `runner`
- указываем что нужны права суперпользователя - `become: true`, `become_method: sudo`
```yml
- name: Install Nginx and create vhosts
  hosts: webservers
  remote_user: runner
  become: true
  become_method: sudo
```
- указываем роль, применяемую к группе хостов `webservers`:
```yml
  roles:
  - nginx-vhosts
```

- добавляем переменные в секции `vars`:
```yml
  vars:
    nginx_sites:
    - "etis.com"
    - "fizfak.ru"
    - "mehmat.ru"
```
3. создаем структуру папок для роли `nginx-vhosts`:
```bash
./roles
    ┗━ nginx-vhosts
        ┣━ handlers
        ┣━ tasks
        ┗━ templates
```

3. создаем шаблон конфигурационного файла site.conf `/templates/site.conf.j2`:
```j2
server {
    listen 80;
    listen [::]:80;

    root /var/www/{{ item }};
    index index.html;

    server_name {{ item }};

    location / { 
        try_files $uri $uri/ =404; 
    }
}
```
4. создаем шаблон файла index.html `/templates/index.html.j2`:
```html
<HTML>
    <BODY>
        <H2>Hello World from {{ item }}</H2>
    <BODY>
</HTML>
```

## Создание роли
добавляем файл с задачами для роли `tasks/main.yml`, в задачах указываем что хотим делать:
1. убеждаемся что установлен Nginx при помощи модуля `ansible.builtin.apt`, при этом 
- обновляем кэш пакетов `update_cache: true`
- версия пакета последняя `state: latest`
```yml
- name: Install Nginx
  ansible.builtin.apt:
    name: nginx
    state: latest
    update_cache: true
```

2. получаем версию Nginx и сохраняем в переменную:
- для получения версии используем `ansible.builtin.shell` с аргументами `nginx -v 2>&1 | jq -R 'split(" ")' | jq -r '.[2]'`
- вывод сохраняем в переменной `nginx_version`
```yml
- name: Get Nginx version
  ansible.builtin.shell: nginx -v 2>&1 | jq -R 'split(" ")' | jq -r '.[2]'
  register: nginx_version
```
3. Выводим значение переменной `nginx_version`:
- при помощи модуля `ansible.builtin.debug` выводим диагностическое сообщение
- указываем параметр msg: `"{{ nginx_version.stdout }}"`
```yml
- name: Print Nginx version
  ansible.builtin.debug:
    msg: "Nginx version: {{ nginx_version.stdout }}"
```
4. копируем конфигурационный файл на хосты при помощи `ansible.builtin.template` в папку `/etc/nginx/conf.d/` 
```yml
- name: Copy nginx.conf from template
  ansible.builtin.template:
    src: 'site.conf.j2'
    dest: '/etc/nginx/conf.d/site-{{ item }}.conf'
    owner: root
    group: root
    mode: '0644'
  loop: '{{ nginx_sites }}'
```
5. создаем папки для каждого из сайтов при помощи `ansible.builtin.file` в папке `/var/www/` 
```yml
- name: Create sites folders
  ansible.builtin.file:
    path: '/var/www/{{ item }}/'
    state: directory
    owner: root
    group: root
    mode: '0755'
  loop: '{{ nginx_sites }}'
```
6. копируем индексную страницу на хосты при помощи `ansible.builtin.template` в папку `/var/www/<имя_сайта>` 
```yml
- name: Copy index from template
  ansible.builtin.template: 
    src: 'index.html.j2'
    dest: '/var/www/{{ item }}/index.html'
    owner: root
    group: root
    mode: '0644'
  loop: '{{ nginx_sites }}'
#  notify: Restart Nginx
```
7. принудительно перезагружаем Nginx при помоищи `ansible.builtin.service` с параметром `name: nginx`, `state: restarted` 
```yml
# instead of Notify nginx
- name: Force ngnix restart
  ansible.builtin.service:
    name: nginx
    state: restarted
```

7. 7a: когда сценарий настроен и выполняется успешно, можно включить handler в `/tasks/main.yml`. Для этого:
- раскомментируем строку `# notify: Restart Nginx` для задачи `Copy index from template`
```yml
- name: Copy index from template
  ansible.builtin.template: 
    src: 'index.html.j2'
    dest: '/var/www/{{ item }}/index.html'
    owner: root
    group: root
    mode: '0644'
  loop: '{{ nginx_sites }}'
  notify: Restart Nginx
```
- добавим или раскомментируем задачу в `handlers/main.yml`:
```yml
---
- name: Restart Nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```
- закомментируем блок задачи `Force ngnix restart` в блоке `# instead of Notify nginx` файла `/tasks/main.yml`  
```yml
# instead of Notify nginx
#- name: Force ngnix restart
#  ansible.builtin.service:
#    name: nginx
#    state: restarted
```

## Проверка

1. на мастер-ноде MASTER (10.0.2.15) запускаем сценарий командой `ansible-playbook playbook.yml -i inventory.ini`  
- ключ `--diff` используем для построчного отображения изменений на целевом хосте  
- либо при помощи скрипта `run.sh`  
2. проверяем командой `curl 10.0.2.16 -H 'Host: etis.com'`, должны получить вывод:
```bash
<HTML>
    <BODY>
        <H2>Hello World from etis.com</H2>
    <BODY>
</HTML>
```
3. также, в браузере из хостовой ОС (предварительно исправив файл hosts: `127.0.0.1 fizfak.ru`) открываем сайт  
`http://fizfak.ru:8080/` (помним, что с прошлых лабораторных настроена трансляция порта 8080:80 для SLAVE)
<img width="551" height="239" alt="image" src="https://github.com/user-attachments/assets/64c4fbb9-754f-455d-80d9-8679a77e92f3" />

