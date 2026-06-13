# # Лаб. работа №4. Ansible roles

создаем инвентарь в `inventory.ini`, указываем один хост SLAVE 10.0.2.16

создаем файл сценария `playbook.yml`, добавляем:
- хосты, на которых выполняем сценарий - группа `webservers`
- под каким пользователем логинимся на целевые хосты - `runner`
- указываем что нужны права суперпользователя - `become: true`, `become_method: sudo`
- указываем роль, применяемую к группе хостов `webservers`

создаем структуру папок для роли `ngins-vhosts`,  
добавляем файл с задачами для роли `tasks: main.yml`, в задачах указываем что хотим делать:
1. убеждаемся что установлен Nginx при помощи модуля `ansible.builtin.apt`, при этом 
- обновляем кэш пакетов `update_cache: true`
- версия пакета последняя `state: latest`
2. получаем версию Nginx и сохраняем в переменную:
- для получения версии используем `ansible.builtin.shell` с аргументами `nginx -v 2>&1 | jq -R 'split(" ")' | jq -r '.[2]'`
- вывод сохраняем в переменной `nginx_version`
3. Выводим значение переменной `nginx_version`:
- при помощи модуля `ansible.builtin.debug` выводим диагностическое сообщение
- указываем параметр msg: `"{{ nginx_version.stdout }}"`

4. создаем шаблон конфигурационного файла site.conf `./roles/nginx-vhosts/templates/site.conf.j2`
5. создаем шаблон файла index.html `./roles/nginx-vhosts/templates/index.html.j2`

6. дописываем задачи в `tasks/main.yml`
- копируем конфигурационный файл при помощи `ansible.builtin.template` в папку `/etc/nginx/conf.d/` 
- создаем папки для каждого из сайтов при помощи `ansible.builtin.file` в папке `/var/www/` 
- копируем индексную страницу при помощи `ansible.builtin.template` в папку `/var/www/<имя_сайта>` 
- принудительно перезагружаем Nginx при помоищи `ansible.builtin.service` с параметром `name: nginx`, `state: restarted` 

7. в файле сценариев `playbook.yml` добавляем переменные в секции `vars`:
```yml
  vars:
    nginx_sites:
    - "etis.com"
    - "fizfak.ru"
    - "mehmat.ru"
```

8. на мастер-ноде запускаем сценарий командой `ansible-playbook playbook.yml -i inventory.ini`
9. проверяем командой `curl 10.0.2.16 -H 'Host: etis.com'`, должны получить вывод:
```bash
<HTML>
    <BODY>
        <H2>Hello World from etis.com</H2>
    <BODY>
</HTML>
```
10. также, в браузере из хостовой ОС (предварительно исправив файл hosts)   
`http://fizfak.ru:8080/` (помним, что с прошлых лабораторных настроена  
трансляция порта 8080:80 для SLAVE)

+++++++++++вставить картинку++++++++++
  

Проверяем playbook на ошибки:  
`ansible-playbook playbook.yml -i inventory.ini --check`

Запускаем playbook из текущей директории:  
`ansible-playbook playbook.yml -i inventory.ini`

