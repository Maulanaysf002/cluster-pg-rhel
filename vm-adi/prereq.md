# ip

10.100.13.94	testing-migrasi-hasura-1
10.100.13.95	testing-migrasi-hasura-2
10.100.13.96	testing-migrasi-hasura-3

root : P@ssw0rd

## cek os

`cat /etc/os-release`

## create user hasura

```bash
useradd -m -s /bin/bash hasura
su -
usermod -aG wheel hasura
id hasura
exit
# login ulang dengan user hasura
```

## cek firewall

## cek versi bawaan etcd, haproxy, keepalived, postgres, patroni 

```bash
dnf repoquery --available etcd
dnf repoquery --available haproxy
dnf repoquery --available keepalived
dnf repoquery --available postgresql-server
dnf repoquery --available patroni
```