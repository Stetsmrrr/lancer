# Установка CoreOS на Bare Metal
***

В данном руководстве представлены инструкции по установке Fedora CoreOS на bare metal тремя различными способами:
* Установка из live ISO
* Установка из PXE
* Установка из контейнера

# Требование
***
Для установки FCOS потребуется файл конфигурации Ignition и место для его размещения (при использовании `python3 -m http.server`). Если файл отсутствует, создайте его. Подробнее смотрите в разделе [создании файла Ignition](https://docs.fedoraproject.org/en-US/fedora-coreos/producing-ign/).

>**Примечание**
>
> Если у вас серверы с различными типами жестких дисков и разным количеством, создайте отдельный настраиваемый файл Ignition для каждого устройства (или класса устройств). Рекомендуется выделять общие части конфигурации в отдельный файл Ignition, который можно включить (через HTTP или inline) в пользовательскую конфигурацию машины.
>
# Установка из live ISO
***
Чтобы установить FCOS на bare metal, используя live ISO в интерактивном режиме, выполните следующие действия:

* Скачайте последний образ ISO со [страницы загрузки](https://fedoraproject.org/coreos/download?stream=stable#baremetal) или с помощью podman (смотрите [документацию](https://coreos.github.io/coreos-installer/cmd/download/) для опций):
  ```
  podman run --security-opt label=disable --pull=always --rm -v .:/data -w /data \
    quay.io/coreos/coreos-installer:release download -s stable -p metal -f iso

 Обратите внимание, что `coreos-installer` используется только в качестве инструмента для загрузки ISO.
 >
    > **Примечание** 
    > 
    >Можете загрузить live ISO также в устаревшем режиме  BIOS или режиме UEFI, независимо от того, какой режим ОС будет использоваться после установки.
    >


* Запишите ISO на диск. На Linux и macOS используйте `dd`. На Windows используйте [Rufus](https://rufus.ie/en/) в режиме "DD Image".
* Загрузите его на целевую систему. Образ ISO позволяет выводить полностью функционирующую систему FCOS без использования дискового хранилища. После загрузки вы получите доступ к командной строке **bash**.
* После запустите `coreos-installer`:
  ~~~
  sudo coreos-installer install /dev/sda \
    --ignition-url https://example.com/example.ign
  ~~~

После завершения установки перезагрузите систему, используя `sudo reboot`. После перезагрузки запустится процесс запуска. В это время Ignition скопирует файлы конфигурации и запустит систему в соответсвии с требованиями.


> **Совет**
> 
> Чтобы узнать больше подробностей про установку Fedora CoreOS используйте команду `coreos-installer install --help`.
>

# Установка из сети
***

> **Примечание** 
>  
> Загрузка образа live PXE требует не менее 2 гигабайта оперативной памяти с аргументом ядра `coreos.live.rootfs_url`, в ином случае 4 гигабайта. Вы также можете установить его на устаревший режим BIOS или режим UEFI, независимо от режима ОС, используемого после установки.
>
## Установка из PXE
Выполните следующие действия для установки образа из PXE:

* Загрузите ядро FCOS PXE, initramfs и образ rootfs:
 ```
 podman run --security-opt label=disable --pull=always --rm -v .:/data -w /data \
  quay.io/coreos/coreos-installer:release download -f pxe
```
  * Следуйте примеру `pxelinux.cfg` для загрузки установки образов с PXELINUX:
```
DEFAULT pxeboot
TIMEOUT 20
PROMPT 0
LABEL pxeboot
    KERNEL fedora-coreos-40.20240504.3.0-live-kernel-x86_64
    APPEND initrd=fedora-coreos-40.20240504.3.0-live-initramfs.x86_64.img,fedora-coreos-40.20240504.3.0-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.1.101:8000/config.ign
IPAPPEND 2
```

## Установка из iPXE
 Устройству, поддерживающему iPXE, потребуется предоставить актуальные скрипты загрузки для извлечения и загрузки артефактов FCOS. 
 
 Следующий пример демонстрирует, как загрузить их напрямую из инфраструктуры Fedora. Для эффективности и надежности рекомендуется отразить их в локальной инфраструктуре, чтобы в дальнейшем настроить `BASEURL` в ссответствии с вашими требованиями.
 ```
 #!ipxe

set STREAM stable
set VERSION 40.20240504.3.0
set INSTALLDEV /dev/sda
set CONFIGURL https://example.com/config.ign

set BASEURL https://builds.coreos.fedoraproject.org/prod/streams/${STREAM}/builds/${VERSION}/x86_64

kernel ${BASEURL}/fedora-coreos-${VERSION}-live-kernel-x86_64 initrd=main coreos.live.rootfs_url=${BASEURL}/fedora-coreos-${VERSION}-live-rootfs.x86_64.img coreos.inst.install_dev=${INSTALLDEV} coreos.inst.ignition_url=${CONFIGURL}
initrd --name main ${BASEURL}/fedora-coreos-${VERSION}-live-initramfs.x86_64.img

boot 
```


# Установка из контейнера
***
Используйте [контейнер](https://quay.io/repository/coreos/coreos-installer) `coreos-installer` из существующей системы для установки на подлключенное блок-устройство. На пример (замените `docker` для `podman`, если потребуется):
```
sudo podman run --pull=always --privileged --rm \
    -v /dev:/dev -v /run/udev:/run/udev -v .:/data -w /data \
    quay.io/coreos/coreos-installer:release \
    install /dev/vdb -i config.ign
```
В данном примере `coreos-installer` загрузит последний стабильный FCOS metal image и установит его в `/dev/vdb`. Затем запустит файл Ignition `config.ign` из текущего каталога в образ. Используйте `--help` для просмотра всех доступных опций.