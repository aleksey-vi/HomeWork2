# HomeWork2
Установите VirtualBox на локальную машину. Установите сам Vagrant: Переходим на https://www.vagrantup.com/downloads.html выбираем соответствующую версию. В данном случае Debian 64-bit и версия 2.2.6. Копируем ссылку и в консоли выполняем:

    curl -O https://releases.hashicorp.com/vagrant/2.2.6/vagrant_2.2.6_x86_64.deb && sudo dpkg -i vagrant_2.2.6_x86_64.deb

После успешного окончания будет установлен Vagrant.
Проверить установку можно командой:

    vagrant -v

## Vagrant

Начальный стенд можно взять отсюда: https://github.com/erlong15/otus-linux В принципе на нем уже можно собрать любой RAID.
В моем репозитории присутствует отредактированный Vagrant-файл, с помощью которого можно собрать любой рейд, для каждого дополнительного диска необходимо добавить в Vagrant-файл следующий блок:

        :sata6 => { :dfile => './sata6.vdi', # Путь, по которому будет создан файл диска 
        :size => 250, # Размер диска в мегабайтах 
        :port => 6 # Номер порта на который будет зацеплен диск 
        },

Обязательно увеличив номер порта и изменив имя файла диска, чтобы исключить дублирование.
При редактировании скаченного Vagrant файла у меня была ошибка ""rsync" could not be found on your PATH. Make sure that rsync
is properly installed on your system and available on the PATH.", которую я исправил пользуясь руководством по ссылке https://qna.habr.com/q/271364, где было необходимо установить плагин vagrant-vbguest:

        vagrant plugin install vagrant-vbguest

и дописать в Vagrant файл 

        config.vm.synced_folder ".", "/vagrant", type: "virtualbox"

Далее подразумеваем, что мы добавили в Vagrantfile 5-ый диск.Добавить в Vagrantfile еще дисков
Далее нужно определиться какого уровня RAID будем собирать. Для это посмотрим, какие блочные устройства у нас есть и исходя из их кол-во, размера и поставленной задачи определимся. Я это делал командой lsblk

## Создание Raid5

При создании реида я пользовался методичкой данной в домашнем задании, а так же хорошим мануалом с пошаговой инструкцией создания RAID-массива по ссылке: https://www.dmosk.ru/miniinstruktions.php?mini=mdadm
и так: 
Занулим на всякий случай суперблоки:

        mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}

И можно создавать рейд следующей командой: 

        mdadm --create --verbose /dev/md0 -l 5 -n 5 /dev/sd{b,c,d,e,f}

>Мы выбрали RAID 5. Опция "-l" - какого уровния RAID создавать.
>Опция "-n" указывает на кол-во устройств в RAID.

Проверим что RAID собрался нормально: 

        cat /proc/mdstat mdadm -D /dev/md0

## Создание конфигурационного файла mdadm.conf

Для того, чтобы быть уверенным что ОС запомнила какой RAID-массив требуется создать и какие компоненты в него входят, создадим файл mdadm.conf Сначала убедимся, что информация верна: 

        mdadm --detail --scan --verbose

### А затем создадим файл mdadm.conf


        echo "DEVICE partitions" > /etc/mdadm/mdadm.conf

        mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf

### Сломать/починить RAID

Сделать это можно, например, искусственно "зафейлив" одно из блочных устройств командой:

        mdadm /dev/md0 --fail /dev/sde

Посмотрим как это отразилось на RAID:

        cat /proc/mdstat mdadm -D /dev/md0

Удалим "сломанный" диск из массива:

        mdadm /dev/md0 --remove /dev/sde

Представим, что мы вставили новый диск в сервер и теперь нам нужно добавить его в RAID. Делается это так:

        mdadm /dev/md0 --add /dev/sde mdadm -D /dev/md0

### Создать GPT раздел, пять партиция и смонтировать их на диск

Создаем раздел GPT на RAID:

        parted -s /dev/md0 mklabel gpt

Создаем партиции:

        parted /dev/md0 mkpart primary ext4 0% 20%

        parted /dev/md0 mkpart primary ext4 20% 40%

        parted /dev/md0 mkpart primary ext4 40% 60%

        parted /dev/md0 mkpart primary ext4 60% 80%

        parted /dev/md0 mkpart primary ext4 80% 100%

Далее, можно создать на этих партициях ФС:

        for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done

И смонтировать их по каталогам:

        mkdir -p /raid/part{1,2,3,4,5}

        for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done

Так же в данном репозитории присутствует скрипт, который, при выполнении в vagrant, будет сам собирать raid, добавлять в автозагрузку, ломать его, чинить и создавать на нём партиции.
При выполнении данного домашнего задания я пользовался следующими ресурсами:<br>
https://www.ibm.com/developerworks/ru/library/l-soft-raid/index.html<br>
https://ru.wikibooks.org/wiki/Mdadm#%D0%9F%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5_%D0%B8%D0%BD%D1%84%D0%BE%D1%80%D0%BC%D0%B0%D1%86%D0%B8%D0%B8_%D0%BE_RAID-%D0%B4%D0%B8%D1%81%D0%BA%D0%B5_%D0%B8_%D0%B5%D0%B3%D0%BE_%D1%80%D0%B0%D0%B7%D0%B4%D0%B5%D0%BB%D0%B0%D1%85<br>
https://gist.github.com/Jekins/2bf2d0638163f1294637<br>
https://qna.habr.com/q/271364<br>
https://habr.com/ru/company/ruvds/blog/325522/<br>
https://losst.ru/napisanie-skriptov-na-bash<br>
https://www.ekzorchik.ru/2015/10/control-of-the-state-of-raid-file/<br>
