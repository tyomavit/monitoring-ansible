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

- Два VPS с Ubuntu 26.04. Серверу A нужно не меньше 8 ГБ памяти (Elasticsearch), серверу B хватит 2 ГБ.
- Два домена (A-записи у регистратора DNS).
- Нода управления (любой Linux-сервер или WSL), с которой Ansible ходит на серверы по SSH.

### 1. Нода: Ansible и ключ

```bash
# ставим Ansible (полный пакет, в нём уже есть нужные коллекции)
apt update && apt install -y ansible git
# проверяем коллекции: ждём три строки с версиями
ansible-galaxy collection list | grep -E 'community.docker|community.general|ansible.posix'
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
# клонируем репозиторий (подставьте свой аккаунт); для публичного можно https://github.com/<аккаунт>/monitoring-ansible.git
git clone git@github.com:<аккаунт>/monitoring-ansible.git
# дальше все команды выполняются отсюда: здесь лежит ansible.cfg
cd monitoring-ansible
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

1. В панели хостинга переустановите оба VPS на Ubuntu 26.04 и добавьте публичный ключ `control_ed25519.pub`.
2. На ноде:

```bash
# забываем старые отпечатки серверов (после переустановки они новые)
ssh-keygen -R <IP сервера A>
ssh-keygen -R <IP сервера B>
# входим по разу, чтобы принять новые отпечатки (отвечаем yes) и сразу выйти
ssh -i ~/.ssh/control_ed25519 root@<IP сервера A> exit
ssh -i ~/.ssh/control_ed25519 root@<IP сервера B> exit
# связь Ansible с обоими серверами: ждём два pong
ansible all -m ping
```

### 7. Установка одной командой

```bash
# все шесть плейбуков по порядку; time покажет, сколько заняла установка
time ansible-playbook site.yml
# повторный прогон: ждём changed=0 на обоих серверах
ansible-playbook site.yml
```

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

## Смена IP сервера

1. Поменять адрес в `inventory.ini`.
2. Поменять A-запись домена у регистратора и дождаться её (`dig +short <домен>`).
3. На ноде: `ssh-keygen -R <старый IP>`, затем один вход по SSH, чтобы принять новый отпечаток.
4. Прогнать `ansible-playbook site.yml`.

Правила файрвола, список целей Prometheus и адрес Logstash для Filebeat обновятся сами: плейбуки берут IP из инвентаря.

---

## Если что-то пошло не так

| Симптом | Куда смотреть |
|---|---|
| Ожидание Elasticsearch дошло до `fatal` | `docker compose logs --tail 40 elasticsearch` на A; частая причина — мало памяти |
| certbot упал | **не перезапускать сразу**: у Let's Encrypt лимит неудачных попыток (около 5 в час). Проверить DNS (`dig`) и что порт 80 открыт |
| Prometheus не видит цели | права на `prometheus/targets/node.yml` должны быть `0644`; `docker compose logs --tail 50 prometheus` |
| Ошибка скрыта надписью `no_log` | временно поставить `no_log: false` у задачи, посмотреть, вернуть `true` |
| `key cannot contain a space` в `.env` | в `.env` попал чужой файл: проверить `dest` у задач `template` |

---

## Известные ограничения

- Логи от B к A идут по интернету без шифрования (порт 5044 без TLS); защита — файрвол по IP.
- При каждой переустановке сервера выпускается новый сертификат: лимит Let's Encrypt около 5 одинаковых сертификатов в неделю на домен.
- Data view в Kibana создаётся вручную (шаг 9).
