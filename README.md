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
* оригинальный Vagrantfile из ДЗ c с модицикацией: увеличено вдвое количество ядер и вчетверо количество памяти, дабы сократить время сборки ядра и модулей.

**Приглашение командной строки хостовой системы выглядит так:**

`$`

**Приглашение командной строки гостевой системы выглядит так:**

`vagrant@$`

## Подготовка
1.1. Установка vagrant

` $ wget https://releases.hashicorp.com/vagrant/2.2.7/vagrant_2.2.7_x86_64.deb `

`$ sudo dpkg -i vagrant_2.2.7_x86_64.deb`

1.2. Установка packer

`$ wget https://releases.hashicorp.com/packer/1.4.4/packer_1.4.4_linux_amd64.zip`

`$ sudo gzip -d packer_1.4.4_linux_amd64.zip -C /usr/local/bin/packer`

`$ sudo chmod +x /usr/local/bin/packer`

1.3. В домашней директории создаю поддиректорию и перехожу в нее

`$ mkdir -p ./otus/hw01 && cd ./otus/hw01`

1.4. Создаю форк репозитория, затем клонирую репозиторий из своего аккаунта и перехожу в директорию

`$ git clone git@github.com:zradeg/manual_kernel_update.git`

`$ cd manual_kernel_update`

## Обновление ядра из исходников

2.1. vagrant up, vagrant ssh, проверяю версию рабочего ядра и ставлю все необходимые пакеты для обновления ядра вручную

`vagrant@$ uname -rs`

>Linux 3.10.0-957.12.2.el7.x86_64

`vagrant@$sudo yum -y install wget vim ncurses-devel make gcc bc bison flex elfutils-libelf-devel openssl-devel grub2`

2.2. Перехожу в директорию, где буду собирать ядра, качаю архив с исходниками, распаковываю и перехожу в директорию с исходниками	

`vagrant@$ cd /usr/src/kernels/`

`vagrant@$ sudo wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.5.2.tar.xz`

`vagrant@$ sudo tar --xz -xvf linux-5.5.2.tar.xz`

`vagrant@$ sudo cd linux-5.5.2/`

2.3. Делаю копию конфига текущего ядра, и применяю конфигурацию его конфигурацию

`vagrant@$ sudo cp -v /boot/config-3.10.0-957.12.2.el7.x86_64 ./.config`

`vagrant@$ sudo make olddefconfig`

2.4. Вручную убираю заведомо лишние модули, т.к. драйвера для ноутбуков и GPU и пр.

`vagrant@$ sudo make menuconfig`

2.5. Приступаю к сборке, использую ключ -j4 для разделения процесса сборки на несколько потоков, чтобы по максимуму использовать выделенные мощности. Собираю непосредственно ядро и упаковываю его, произвожу сборку модулей, устанавливаю все по очереди, не забывая про хедеры, чтобы встали vb guest additions

`vagrant@$ sudo make -j4 bzImage`

`vagrant@$ sudo make -j4 modules`
 
`vagrant@$ sudo make -j4`
 
`vagrant@$ sudo make -j4 modules_install`
 
`vagrant@$ sudo make -j4 headers_install`
 
`vagrant@$ sudo make -j4 install`

>VirtualBox Guest Additions: Building the modules for kernel 5.5.2.

>VirtualBox Guest Additions: Look at /var/log/vboxadd-setup.log to find out what went wrong

2.6. Проверяю, установились ли vb guest additions

`vagrant@$ sudo lsmod | grep vbox`

>vboxsf                 57344  1
vboxguest             356352  2 vboxsf

2.7. Обновляю загрузчик и перезагружаюсь

`vagrant@$ sudo grub2-mkconfig -o /boot/grub2/grub.cfg`

`vagrant@$ sudo grub2-set-default 0`

`vagrant@$ sudo reboot`

2.8. Очередной vagrant ssh и проверяю версию ядра

`vagrant@$ uname -rs`

>Linux 5.5.2

Бинго!

2.9. Из командной строки хостовой машины создаю общую папку со ссылкой на диск D:, предварительно выяснив имя виртуалки, с которой работаю посредством VBoxManage list runningvms

`$ VBoxManage sharedfolder add manual_kernel_update_kernel-update_1581347777248_21739 --name disk_d --hostpath d:\`

2.10. В виртуалке удаляю старое ядро и компоненты, также удаляю исходники свежеустановленного ядра

`vagrant@$ sudo yum erase kernel-3.10.0`

`vagrant@$ sudo yum erase kernel-3.10.0 kernel-headers.x86_64 kernel-devel-3.10.0`

`vagrant@$ sudo rm -rf /usr/src/kernels/linux-5.5.2`

2.11. Создаю точку монтирования для общей папки, монтирую ее, проверяю результат

`vagrant@$ sudo mkdir /mnt/d`

`vagrant@$ sudo mount -t vboxsf disk_d /mnt/d`

`vagrant@$ df -h`

>Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        2.0G     0  2.0G   0% /dev
tmpfs           2.0G     0  2.0G   0% /dev/shm
tmpfs           2.0G  8.4M  2.0G   1% /run
tmpfs           2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/sda1        40G  5.7G   35G  15% /
tmpfs           394M     0  394M   0% /run/user/1000
disk_d          688G  315G  373G  46% /mnt/d

## Загрузка образа в vagramt cloud и проверка

3.1. Для уменьшения объема образа, а так же для тогоа, чтобы предотвратить возникновение ошибки "Warning: Authentication failure. Retrying..." и невозможности залогиниться без пароля по ssh-ключу, запускаю следующий скрипт:

`vagrant@$ sudo yum update -y`

`vagrant@$ sudo yum clean all`

`vagrant@$ sudo mkdir -pm 700 /home/vagrant/.ssh`

`vagrant@$ sudo curl -sL https://raw.githubusercontent.com/mitchellh/vagrant/master/keys/vagrant.pub -o /home/vagrant/.ssh/authorized_keys`

`vagrant@$ sudo chmod 0600 /home/vagrant/.ssh/authorized_keys`

`vagrant@$ sudo chown -R vagrant:vagrant /home/vagrant/.ssh`

`vagrant@$ sudo rm -rf /tmp/*`

`vagrant@$ sudo rm  -f /var/log/wtmp /var/log/btmp`

`vagrant@$ sudo rm -rf /var/cache/* /usr/share/doc/*`

`vagrant@$ sudo rm -rf /var/cache/yum`

`vagrant@$ sudo rm -rf /vagrant/home/*.iso`

`vagrant@$ sudo rm  -f ~/.bash_history`

`vagrant@$ sudo history -c`

`vagrant@$ sudo rm -rf /run/log/journal/*`

`vagrant@$ sudo dd if=/dev/zero of=/EMPTY bs=1M`

`vagrant@$ sudo rm -f /EMPTY`

3.2. Средствами vagrant собираю образ, подключаю его в БД vagrant и заливаю в облако, предварительно залогинившись

`$ vagrant package --base manual_kernel_update_kernel-update_1581347777248_21739 --output centos-7-5-manual_kernel_update_v2.box`

`$ vagrant box add --name manual_kernel_update centos-7-5-manual_kernel_update_v2.box`

`$ vagrant cloud publish --release zradeg/centos-7-5-manual_kernel_update_v2 1.0 virtualbox centos-7-5-manual_kernel_update_v2.box`

3.3. Исправляю Vagrantfile, вернув назад количество ядер и объем выделенной памяти, а также непосредственно ссылку на образ, переношу Vagrantfile на другую машину, запускаю и проверяю версию ядра и наличие vb guest additions

`vagrant@$ # uname -rs`

>Linux 5.5.2

`vagrant@$ sudo lsmod | grep vbox`

>vboxsf                 57344  0
vboxguest             356352  2 vboxsf

## Подведение итога

* Установлено новое ядро из исходника версии 5.5.2
* Установлен vbox guest additions - есть возможность монтировать общие папки
* Образ сжат и загружен в облако vagrant
* При скачивании на другой машине отрабатывает vagrant up с вытягиванием образа из облака и развертыванием виртуальной системы с новым ядром, сохраняя возможность беспарольного подключения по ssh-ключу

