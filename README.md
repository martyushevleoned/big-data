# Репозиторий с лабораторными работами
* Студент: Мартюшев Л.А.
* Группа: РИ-411050
* Дисциплина: Методы и алгоритмы анализа больших данных

## Выполненные ЛР
1) [Лабораторная работа №1 - CVE](lab1/README.md)
2) [Лабораторная работа №2 - Развёртывание кластера БД](lab2/README.md)
3) [Лабораторная работа №3 - Трафик к БД]()

## Подготовка к ЛР
Установка Docker:
```shell
wget --output-document=/tmp/get-docker.sh https://get.docker.com
sudo sh /tmp/get-docker.sh
```
Добавление в группу:
```shell
sudo groupadd docker
sudo usermod -aG docker $USER
reboot
```
Установка psql
```shell
sudo apt install postgresql-client
```