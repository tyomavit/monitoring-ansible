# monitoring-ansible

Развёртывание учебного стенда мониторинга на двух VPS одной командой Ansible.

| Сервер | Группа | Что работает | Адрес снаружи |
|---|---|---|---|
| A, `elk-vps` | `elk` | Elasticsearch, Logstash, Kibana, Filebeat, node-exporter | `https://kibana.artemdevops.ru` |
| B, `monitoring-vps` | `monitoring` | Prometheus, Grafana (с дашбордом Node Exporter Full), Filebeat, node-exporter | `https://grafana.artemdevops.ru` |

Логи обоих серверов: journald → Filebeat → Logstash на A (порт 5044) → Elasticsearch → Kibana.
Метрики обоих серверов: node-exporter → Prometheus на B (опрос раз в 15 секунд) → Grafana.
Снаружи открыты только 22, 80, 443; Kibana и Grafana доступны через nginx с сертификатами Let's Encrypt.

---

## Структура проекта

| Путь | Что там |
|---|---|
| `site.yml` | общий плейбук: вызывает все остальные по порядку |
| `requirements.yml` | коллекции Ansible, которые нужны плейбукам |
| `prep.yml` | имя сервера, уникальный machine-id, swap, параметры ядра, вход только по ключу |
| `docker.yml` | Docker и плагин Compose |
| `firewall.yml` | ufw: общие правила и порты между A и B |
| `elk.yml` | стек A: файлы, запуск Elasticsearch, пользователи, `filebeat setup`, запуск остального |
| `monitoring.yml` | стек B: файлы, список целей Prometheus, запуск |
| `nginx.yml` | nginx, сертификат Let's Encrypt, конфиг сайта |
| `inventory.ini` | серверы и их IP (**не в git**, образец `inventory.ini.example`) |
| `group_vars/all/vars.yml` | версии, домены, ярлыки на секреты |
| `group_vars/all/vault.yml` | секреты, зашифрованы ansible-vault (**не в git**, образец `vault.yml.example`) |
| `group_vars/elk.yml`, `group_vars/monitoring.yml` | домен сайта и адрес приложения для nginx на каждом сервере |
| `files/` | файлы, которые едут на серверы без изменений |
| `templates/` | шаблоны: `.env` стеков, список целей Prometheus, конфиг nginx |
| `.vault_pass` | пароль от сейфа (**не в git**, только на ноде) |

---

## Развёртывание с нуля

### 0. Что нужно заранее

Три машины с Ubuntu 26.04: новые или существующие, которые можно переустановить (все данные на них будут стёрты).

| Машина | Роль | CPU | RAM | Диск | Примечание |
|---|---|---|---|---|---|
| `ansible-ctl` | нода управления: отсюда Ansible ходит на серверы | 1 | 2 ГБ | 10 ГБ | подойдёт и WSL на ноутбуке |
| `elk-vps` (A) | логи: ELK | 2 | **8 ГБ** | 20 ГБ | Elasticsearch занимает около половины RAM; логи хранятся 14 дней |
| `monitoring-vps` (B) | метрики: Prometheus и Grafana | 2 | 2 ГБ | 20 ГБ | метрики хранятся 15 дней |

Ещё нужны два домена (A-записи у регистратора DNS) и, по желанию, аккаунт на GitHub с этим репозиторием.

При создании машин в панели хостинга сразу называйте их по ролям, как в таблице: по этим именам их проще различать в панели и в выводе Ansible.

### 1. Нода: имя, Ansible и ключ

```bash
# имя ноды по роли (у свежего VPS оно случайное); видно в приглашении после перезахода
hostnamectl set-hostname ansible-ctl
# ставим Ansible и git
apt update && apt install -y ansible git
# ключ, которым нода входит на серверы (если его ещё нет)
ssh-keygen -t ed25519 -f ~/.ssh/control_ed25519 -C ansible-ctl
# публичная часть ключа: её добавляем на серверы
cat ~/.ssh/control_ed25519.pub
```

### 2. Проект

Публичный репозиторий клонируется по HTTPS без ключей. Для клонирования по SSH (приватный репозиторий или чтобы делать `git push` с ноды) сначала добавьте ключ ноды на GitHub: Settings → SSH and GPG keys → New SSH key.

```bash
# (для SSH) ключ ноды для GitHub и его публичная часть, которую вставляем на GitHub
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C ansible-ctl-github
cat ~/.ssh/id_ed25519.pub
# проверяем доступ: ждём "Hi <аккаунт>! You've successfully authenticated"
ssh -T git@github.com
# (для коммитов с ноды) подпись автора коммитов: имя и почта, как на GitHub
git config --global user.name "<Имя Фамилия>"
git config --global user.email "<почта на GitHub>"
# клонируем репозиторий (подставьте свой аккаунт); для публичного можно https://github.com/<аккаунт>/monitoring-ansible.git
git clone git@github.com:<аккаунт>/monitoring-ansible.git
# дальше все команды выполняются отсюда: здесь лежит ansible.cfg
cd monitoring-ansible
# свежие версии коллекций, которые нужны плейбукам (Docker, ufw, sysctl и mount)
ansible-galaxy collection install -r requirements.yml --upgrade
# проверяем: ждём три строки с версиями
ansible-galaxy collection list | grep -E 'community.docker|community.general|ansible.posix'
```

### 3. Инвентарь

```bash
# рабочий инвентарь из образца
cp inventory.ini.example inventory.ini
# вписываем реальные IP вместо 203.0.113.x
vim inventory.ini
```

Это **единственное место**, где записаны IP серверов. Правила файрвола, список целей Prometheus и адрес Logstash для Filebeat плейбуки берут отсюда.

### 4. Секреты

```bash
# пароль от сейфа: случайная строка сразу в файл, на экран не выводится
openssl rand -hex 16 > .vault_pass
# читать файл может только владелец
chmod 600 .vault_pass
# новые файлы в этой оболочке создаются с правами только для владельца
umask 077
# собираем файл секретов со случайными значениями (на экран они не выводятся)
{
  echo '---'
  for v in vault_elastic_password vault_kibana_password vault_logstash_internal_password vault_grafana_admin_password; do
    echo "$v: $(openssl rand -hex 16)"
  done
  echo "vault_kibana_encryption_key: $(openssl rand -hex 32)"
  echo "vault_certbot_email: CHANGE_ME"
} > group_vars/all/vault.yml
# вписываем свою почту для Let's Encrypt вместо CHANGE_ME
vim group_vars/all/vault.yml
# шифруем файл (пароль Ansible берёт из .vault_pass)
ansible-vault encrypt group_vars/all/vault.yml
# проверяем: первая строка $ANSIBLE_VAULT;1.1;AES256
head -1 group_vars/all/vault.yml
# имена переменных без значений: ждём шесть vault_...
ansible-vault view group_vars/all/vault.yml | cut -d: -f1
# git игнорирует все три файла: ждём три строки с правилами
git check-ignore -v .vault_pass inventory.ini group_vars/all/vault.yml
```

Сохраните `.vault_pass` в менеджер паролей: без него сейф не открыть.

### 5. DNS

У регистратора: A-запись `kibana.artemdevops.ru` → IP сервера A, `grafana.artemdevops.ru` → IP сервера B.

```bash
# оба имени отвечают нужными IP (если пусто, подождите 5–10 минут)
dig +short kibana.artemdevops.ru grafana.artemdevops.ru
```

### 6. Серверы

#### 6.1 Создать или переустановить

В панели хостинга создайте оба VPS (или переустановите существующие) на Ubuntu 26.04 и **в той же форме выберите публичный ключ ноды** (`control_ed25519.pub`, его выводит команда из шага 1). Ключ из панели попадает на сервер только при создании или переустановке: добавленный в аккаунт позже на работающий сервер сам не попадёт.

Если панель не принимает ключ («Некорректный ключ»):
- ключ должен быть **одной строкой**: `ssh-ed25519 AAAA... ansible-ctl`, без переносов;
- попробуйте вставить его без комментария в конце (`ansible-ctl`);
- посмотрите, какого типа ключ панель уже приняла раньше (первое слово строки: `ssh-ed25519` или `ssh-rsa`). Если панель принимает только RSA, создайте для ноды ключ RSA: `ssh-keygen -t rsa -b 4096 -f ~/.ssh/control_rsa -C ansible-ctl` и поменяйте путь к ключу в `inventory.ini` (`ansible_ssh_private_key_file`).

#### 6.2 Проверить вход по ключу

```bash
# забываем старые отпечатки серверов (после переустановки они новые; для новых серверов ответ "not found" нормален)
ssh-keygen -R <IP сервера A>
ssh-keygen -R <IP сервера B>
# входим по разу строго по ключу, принимаем новый отпечаток (yes); ждём выход без вопроса о пароле
ssh -i ~/.ssh/control_ed25519 -o PasswordAuthentication=no root@<IP сервера A> exit
ssh -i ~/.ssh/control_ed25519 -o PasswordAuthentication=no root@<IP сервера B> exit
# связь Ansible с обоими серверами: ждём два pong
ansible all -m ping
```

**Не запускайте `site.yml`, пока вход по ключу не проходит:** `prep.yml` запрещает вход по паролю, и без рабочего ключа сервер станет недоступен.

#### 6.3 Если ключа на сервере нет: запасные способы

`Permission denied (publickey)` или вопрос о пароле значат, что сервер не знает ключ ноды. Вопрос о пароле появляется, даже если вход по паролю в панели выключен: так SSH-клиент пробует последний способ входа, но никакой пароль сервер не примет. Варианты, от более частого к редкому:

**А. С машины, которой сервер уже доверяет** (например, с ноутбука, чей ключ выбран в панели). Записи в SSH config не нужны: путь к ключу указывается прямо в команде.

```bash
# (нода) публичная часть ключа ноды: одна строка, копируем целиком
cat ~/.ssh/control_ed25519.pub
# (ноутбук) дописываем её на сервер; -i путь к ключу ноутбука; два знака >>, иначе сотрётся ключ ноутбука
ssh -i ~/.ssh/<ключ ноутбука> root@<IP сервера> "echo '<строка ключа ноды>' >> ~/.ssh/authorized_keys"
```

**Б. По паролю root.** Включите вход по паролю в панели (обычно с перезагрузкой сервера), возьмите пароль root в панели или в письме о создании сервера, затем:

```bash
# дописываем ключ ноды на сервер (один раз спросит пароль root)
ssh-copy-id -i ~/.ssh/control_ed25519.pub root@<IP сервера>
```

После проверки входа по ключу выключите пароль в панели обратно (`prep.yml` всё равно запретит его в SSH).

**В. Через веб-консоль хостинга (VNC)**, если не работает ничего из выше: войти в консоль сервера в браузере и вписать строку ключа в `/root/.ssh/authorized_keys` вручную.

### 7. Установка одной командой

```bash
# все шесть плейбуков по порядку; time покажет, сколько заняла установка
time ansible-playbook site.yml
# повторный прогон: ждём changed=0 на обоих серверах
ansible-playbook site.yml
```

Для ориентира: первый прогон занимает около 6–7 минут (`changed` около 30 на сервер), повторный около полутора минут с `changed=0`. Строки `[DEPRECATION WARNING]` от модулей коллекций (sysctl, mount) не ошибка: это предупреждения авторов коллекций, они уходят после `ansible-galaxy collection install -r requirements.yml --upgrade`, когда авторы выпускают исправление.

### 8. Проверки

```bash
# контейнеры A: все Up, без перезапусков
ansible elk -m command -a "chdir=/opt/monitoring-compose/elk docker compose ps"
# контейнеры B
ansible monitoring -m command -a "chdir=/opt/monitoring-compose/prometheus-grafana docker compose ps"
# цели Prometheus: ждём три "health":"up" (через минуту после запуска)
ansible monitoring -m shell -a "curl -s http://127.0.0.1:9090/api/v1/targets | grep -o '\"scrapeUrl\":\"[^\"]*\"\|\"health\":\"[^\"]*\"'"
# с B к Logstash на A есть соединение: ждём IP сервера B
ansible elk -m command -a "ss -tn state established ( sport = :5044 )"
# HTTPS отвечает, HTTP перенаправляет
curl -sI https://kibana.artemdevops.ru | head -1
curl -sI http://grafana.artemdevops.ru | head -1
# пробное продление сертификатов на обоих серверах
ansible all -m command -a "certbot renew --dry-run"
```

В браузере:

- **Grafana** (`https://grafana.artemdevops.ru`): логин `admin`, пароль `vault_grafana_admin_password`. Дашборд Node Exporter Full уже загружен, в списке Nodename два сервера.
- **Kibana** (`https://kibana.artemdevops.ru`): логин `elastic`, пароль `vault_elastic_password`. Логи обоих серверов видны в Discover (All logs), поле `host.name`.

```bash
# посмотреть пароль из сейфа (только на своём экране)
ansible-vault view group_vars/all/vault.yml | grep -E 'grafana_admin|elastic_password'
```

### 9. Ручной шаг после установки

Kibana: Stack Management → Data Views → Create data view, шаблон `filebeat-*`, поле времени `@timestamp`.

---

## Шаги, если сменился IP сервера

1. Поменять адрес в `inventory.ini`.
2. Поменять A-запись домена у регистратора и дождаться её:

```bash
# имя отвечает новым IP (если старый, подождите 5–10 минут)
dig +short <домен>
```

3. На ноде заменить отпечаток сервера:

```bash
# забываем отпечаток старого адреса
ssh-keygen -R <старый IP>
# входим по новому адресу строго по ключу, принимаем отпечаток (yes) и выходим
ssh -i ~/.ssh/control_ed25519 -o PasswordAuthentication=no root@<новый IP> exit
```

4. Прогнать установку:

```bash
# плейбуки возьмут новый IP из инвентаря
ansible-playbook site.yml
```

Список целей Prometheus и адрес Logstash для Filebeat обновятся сами: плейбуки берут IP из инвентаря, а Compose пересоздаст Filebeat на B, когда в его `.env` поменяется адрес.

Файрвол ведёт себя иначе: плейбук **добавит** правило для нового IP, но правило для старого адреса останется. Если сменился IP одного сервера, на **другом** сервере удалите устаревшее правило вручную:

```bash
# правила с номерами: ищем строку со старым IP
ansible all -m command -a "ufw status numbered"
# удаляем правило по номеру на нужном сервере (подставьте группу и номер; --force без вопроса y/n)
ansible <elk или monitoring> -m command -a "ufw --force delete <номер>"
```

---

## Если что-то пошло не так

### Где что лежит

| Сервер | Группа Ansible | Папка стека | Сервисы (имена для `docker compose`) |
|---|---|---|---|
| A | `elk` | `/opt/monitoring-compose/elk` | `elasticsearch`, `kibana`, `logstash`, `filebeat`, `node-exporter` |
| B | `monitoring` | `/opt/monitoring-compose/prometheus-grafana` | `prometheus`, `grafana`, `filebeat`, `node-exporter` |

Команды `docker compose` работают **только из папки стека**: в другом месте будет ошибка `no configuration file provided: not found`.

### Логи и состояние: с ноды, не заходя на сервер

```bash
# состояние контейнеров A (для B: группа monitoring и папка prometheus-grafana)
ansible elk -m command -a "chdir=/opt/monitoring-compose/elk docker compose ps"
# последние 50 строк лога сервиса (подставьте группу, папку и сервис из таблицы выше)
ansible monitoring -m command -a "chdir=/opt/monitoring-compose/prometheus-grafana docker compose logs --tail 50 prometheus"
# только строки с ошибками и предупреждениями
ansible monitoring -m shell -a "cd /opt/monitoring-compose/prometheus-grafana && docker compose logs --tail 200 grafana | grep -iE 'error|warn'"
# перезапуск одного сервиса (данные в томах сохраняются)
ansible monitoring -m command -a "chdir=/opt/monitoring-compose/prometheus-grafana docker compose restart grafana"
```

То же самое на самом сервере (после `ssh -i ~/.ssh/control_ed25519 root@<IP>`):

```bash
# переходим в папку стека
cd /opt/monitoring-compose/prometheus-grafana
# состояние и лог сервиса
docker compose ps
docker compose logs --tail 50 prometheus
```

### Частые симптомы

| Симптом | Причина и что делать |
|---|---|
| Ожидание Elasticsearch дошло до `fatal` | лог `elasticsearch` на A; частая причина — мало памяти |
| certbot упал | **не перезапускать сразу**: у Let's Encrypt лимит неудачных попыток (около 5 в час). Проверить DNS (`dig +short <домен>`) и что порт 80 открыт (`ufw status`) |
| Prometheus не видит цели | права на `prometheus/targets/node.yml` должны быть `0644` (Prometheus в контейнере работает не от root); лог `prometheus` на B |
| Дашборд Grafana или Kibana пустой, в поле Datasource (или data view) непонятный ID | страница открыта по старой ссылке или вкладке от прошлой установки: выбранные значения хранятся прямо в адресе. Открыть сайт с главной страницы и выбрать дашборд из списка. Если и так пусто — сверить uid источника в Grafana (команда ниже) с uid в JSON дашборда |
| Ошибка скрыта надписью `no_log` | временно поставить `no_log: false` у задачи, посмотреть, вернуть `true` |
| `key cannot contain a space` в `.env` | в `.env` попал чужой файл: проверить `dest` у задач `template` |
| `no configuration file provided: not found` | команда `docker compose` запущена не из папки стека (см. «Где что лежит») |

```bash
# uid источников данных в Grafana (пароль admin из сейфа подставляется на ноде; ждём "uid":"prometheus")
ansible monitoring -m shell -a "curl -s -u admin:{{ grafana_admin_password }} http://127.0.0.1:3000/api/datasources | grep -o '\"uid\":\"[^\"]*\"'"
```

---

## Известные ограничения

- Логи от B к A идут по интернету без шифрования (порт 5044 без TLS); защита — файрвол по IP.
- При каждой переустановке сервера выпускается новый сертификат: лимит Let's Encrypt около 5 одинаковых сертификатов в неделю на домен.
- Data view в Kibana создаётся вручную (шаг 9).
