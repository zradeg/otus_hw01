# С чего начинается Linux
## Домашнее задание
### Уровень: Base

>Обновить ядро в базовой системе
>Цель: Студент получит навыки работы с Git, Vagrant, Packer и публикацией готовых образов в Vagrant Cloud.
>В материалах к занятию есть методичка, в которой описана процедура обновления ядра из репозитория. По данной методичке требуется выполнить необходимые действия. Полученный в ходе выполнения ДЗ Vagrantfile должен быть залит в ваш репозиторий. Для проверки ДЗ необходимо прислать ссылку на него.

## Рабочее окружение:
* Windows 10.0.18362
* vagrant 2.2.7
* Oracle VirtualBox 5.2.36 r135684
* packer 1.5.1
* github: https://github.com/zradeg

**Приглашение командной строки хостовой системы выглядит так:**

```$```

**Приглашение командной строки гостевой системы выглядит так:**

```vagrant@$```

## Подготовка
1. Установка vagrant

```$ wget https://releases.hashicorp.com/vagrant/2.2.7/vagrant_2.2.7_x86_64.deb```
```$ sudo dpkg -i vagrant_2.2.7_x86_64.deb```

2. Установка packer

```$ wget https://releases.hashicorp.com/packer/1.4.4/packer_1.4.4_linux_amd64.zip```
```$ sudo gzip -d packer_1.4.4_linux_amd64.zip -C /usr/local/bin/packer```
```$ sudo chmod +x /usr/local/bin/packer```

3. В домашней директории создаю поддиректорию и перехожу в нее

```$ mkdir -p ./otus/hw01 && cd ./otus/hw01```

3. Создаю форк репозитория, затем клонирую репозиторий из своего аккаунта и перехожу в директорию

```$ git clone git@github.com:zradeg/manual_kernel_update.git```
```$ cd manual_kernel_update```

4. Поднимаю виртуалку, подключаюсь, проверяю текущую версию ядра, произвожу обновление пакетов из репозитория:

```$ vagrant up```
```$ vagrant ssh```
```vagrant@$ uname -r```
> 3.10.0-957.12.2.el7.x86_64

```vagrant@$ sudo yum -y update```

5. Подключаю репозиторий **epel**, запускаю установку ядра версии *ml*:

```vagrant@$ sudo yum install -y http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm```
```vagrant@$ sudo yum --enablerepo elrepo-kernel install kernel-ml -y```

6. Вношу изменения в конфигурацию загрузчика, определяю новое ядро дефолтным при старте системы, перезагружаю виртуалку:

```vagrant@$ sudo grub2-mkconfig -o /boot/grub2/grub.cfg```
```vagrant@$ sudo grub2-set-default 0```
```vagrant@$ sudo reboot```

7. Захожу в гостевой хост, проверяю версию ядра

```$ vagrant up```
```$ vagrant ssh```
```vagrant@$ uname -r```
> 5.5.2-1.el7.elrepo.x86_64

8. Удаляю старое ядро и его компоненты, ставлю компоненты к новому ядру, выключаю гостевой хост:

```vagrant@$ sudo yum erase kernel-3.10.0 kernel-headers-3.10.0 kernel-devel -3.10.0```
```vagrant@$ sudo yum --enablerepo elrepo-kernel install kernel-ml-headers kernel-ml-devel gcc -y```

9. Cредствами virtualbox настраиваю подключение общих папок, подключаю виртуальный cd с VirtualBox Guest Additions, снова поднимаю хост, захожу, запускаю установку vbguestadds и подмонтирую общую папку :

```$ vagrant up```
```$ vagrant ssh```
```vagrant@$ sudo mkdir /mnt/cdrom && sudo mkdir /mnt/d```
```vagrant@$ sudo mount /dev/cdrom /mnt/cdrom```
```vagrant@$ sudo /mnt/cdrom/VBoxLinuxAdditions.run```
```vagrant@$ sudo mount -t vboxsf D_DRIVE /mnt/d```

10. Создаю новый образ средствами vagrant

```$ vagrant package --base 'manual_kernel_update_kernel-update_1580998659941_3141' --output centos-7-5-kernel_update_by_yum```

11. Полученный образ добавляю в db vagrant, проверяю наличие, заменяю имя образа в Vagrantfile, поднимаю на его основе гостевой хост, захожу в него

```$ vagrant box add --name kernel_update_by_yum centos-7-5-kernel_update_by_yum```
```$ vagrant box list
>centos/7             (virtualbox, 1905.1)
centos-7-5-kernel_update_by_yum (virtualbox, 0)

```$ vagrant up```
```$ vagrant ssh```

```vagrant@$ uname -r```
> 5.5.2-1.el7.elrepo.x86_64

12. Заливаю новый образ в vagrant cloud
https://app.vagrantup.com/zradeg/boxes/centos-7-5-kernel_update_by_yum