# Домашнее задание: LDAP — централизованная авторизация и аутентификация

## Текст задания

1. Установить FreeIPA.
2. Написать Ansible-playbook для конфигурации клиента.
3. (*) Настроить аутентификацию по SSH-ключам.
4. (**) Firewall должен быть включен на сервере и на клиенте.

Стенд развёрнут на Proxmox.

## Схема сети

```
Локальная машина
   │
   ├── FreeIPA-сервер: ipa.otus.lan   (192.168.1.170, VM 550, Ubuntu 22.04)
   │        домен otus.lan / realm OTUS.LAN, без интегрированного DNS (--no-dns)
   │
   └── Клиент: client1.otus.lan      (192.168.1.171, VM 551)
            ipa-client-install --mkhomedir, firewall (ufw) включён
```

## Установка FreeIPA-сервера

Особенность стенда: в Ubuntu 24.04 (noble) пакет `freeipa-server` удалён из архива
(остался только клиентский стек sssd), PPA freeipa не поддерживает noble. Поэтому
сервер развёрнут на Ubuntu 22.04 (jammy, freeipa-server 4.9.x из universe).

Установка:

```
apt install -y freeipa-server
ipa-server-install \
  --unattended \
  --domain=otus.lan --realm=OTUS.LAN \
  --ds-password=<ds_pass> --admin-password=<admin_pass> \
  --no-dns --no-ntp
```

Предварительно: `hostnamectl set-hostname ipa.otus.lan`, записи в /etc/hosts
(нет DNS): `192.168.1.170 ipa.otus.lan ipa`, `192.168.1.171 client1.otus.lan client1`.

Проверка kerberos:

```
# kinit admin
# klist
Ticket cache: KEYRING:persistent:0:0
Default principal: admin@OTUS.LAN
```

## Ansible плейбук клиента

`ansible/provision.yml` (инвентарь `ansible/hosts`): устанавливает тулзы, ufw с
правилами FreeIPA (порт 88/464/80/443/389/636/123), freeipa-client и выполняет
вход в домен:

```
ipa-client-install --mkhomedir --domain=otus.lan --server=ipa.otus.lan \
  --no-ntp --principal=admin --password=<admin_pass> --unattended
```

Проверка после join: `kinit admin` + `klist` → билет `krbtgt/OTUS.LAN`.

## (*) Аутентификация по SSH-ключам

Через FreeIPA хранится sshpubkey пользователя, OpenSSH клиента берёт ключ
из каталога (sss_ssh_authorizedkeys) — authorized_keys на хостах не нужны.

```
# на сервере:
ipa user-add otususer --first=Otus --last=User --password
ipa user-mod otususer --sshpubkey="ssh-ed25519 AAAA... comment"

# на клиенте:
sss_ssh_authorizedkeys otususer   # возвращает ключ
ssh otususer@client1.otus.lan     # вход без пароля, по ключу из IPA
```

## (**) Firewall

ufw включён на сервере и клиенте (default deny incoming), разрешены порты FreeIPA:
kerberos 88 tcp/udp, kadmin/kpasswd 464 tcp/udp, http 80, https 443, ldap 389,
ldaps 636, ntp 123, ssh 22.

## Особенности и заметки

- Ubuntu 24.04 не подходит для IPA-сервера — использовать jammy или RHEL-family.
- `--no-dns`: без интегрированного DNS развязка имён через /etc/hosts на обоих хостах.
- `--no-ntp`: время синхронизировано от хоста виртуализации; в проде обязателен NTP
  (Kerberos чувствителен к рассинхрону >5 мин).
- Клиент требует реверс-соответствия hostname; перед install проверять
  `hostname -f` → client1.otus.lan.
- Повторный запуск плейбука идемпотентен: проверяется наличие /etc/krb5.keytab.

## Файлы

| Файл | Назначение |
|---|---|
| `ansible/hosts` | инвентарь (ipa, client1) и переменные домена |
| `ansible/provision.yml` | конфигурация клиента: тулзы, firewall, join, kinit-проверка |
| `README.md` | этот файл |
