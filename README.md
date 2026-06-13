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
- для получения версии используем `ansible.builtin.shell` с аргументами `nginx -v 2>&1 | jq -R 'split(" ")' | jq .[2]`
- вывод сохраняем в переменной `nginx_version`
3. Выводим значение переменной `nginx_version`:
- при помощи модуля `ansible.builtin.debug` выводим диагностическое сообщение
- указываем параметр msg: `"{{ nginx_version.stdout }}"`


Проверяем playbook на ошибки:  
`ansible-playbook playbook.yml -i inventory.ini --check`

Запускаем playbook из текущей директории:  
`ansible-playbook playbook.yml -i inventory.ini`

