# Установка CoreOS на Bare Metal
***

В данном гайде представлены инструкции по установке Fedora CoreOS на bare metal тремя различными способами:
* Установка из live ISO
* Установка из PXE
* Установка из контейнера

# Требование
***
Для установки FCOS потребуется файл конфигурации Ignition и место для его размещения (при использовании `python3 -m http.server`). Если файл отсутствует, обратитесь к [созданию файла Ignition](https://docs.fedoraproject.org/en-US/fedora-coreos/producing-ign/).

>**Примечание**
>
> Если у вас серверы с различными типами и\или количеством жестких дисков, создайте отдельный настраиваемый Ignition конфиг для каждого устройства (или класса устройств). Хорошим примером является наличие общих частей конфигурации выделенных в отдельную Ignition конфигурацию, способную объединиться (через HTTP или inline) в настраиваемый на уровне машины конфиг.
>
# Установка из live ISO
***
Чтобы установить FCOS на bare metal, используя live ISO в интерактивном режиме, выполните следующие действия:

* Скачайте последний образ ISO со [страницы загрузки](https://fedoraproject.org/coreos/download?stream=stable#baremetal) или с помощью podman (смотрите [документацию](https://coreos.github.io/coreos-installer/cmd/download/) для опций):
  ```
  podman run --security-opt label=disable --pull=always --rm -v .:/data -w /data \
    quay.io/coreos/coreos-installer:release download -s stable -p metal -f iso
    ```
 Помните, это всего лишь использование `coreos-installer` в качестве инструмента для загрузки ISO.
 >
    > **Примечание** 
    > 
    >Можете загрузить live ISO также в легаси  BIOS или режим UEFI, независимо от того, какой режим ОС будет использоваться после установки.
    >


* Запишите ISO на диск. На Linux и macOS используйте `dd`. На Windows используйте [ Rufus](https://rufus.ie/en/) в режиме "DD Image".
* Загрузите его на целевую систему. Образ ISO позволяет выводить полностью функционирующую систему FCOS исключительно из памяти (т.е. без использования дискового хранилища). После загрузки вы получите доступ к командной строке bash.
* После запустите `coreos-installer`:
  ~~~
  sudo coreos-installer install /dev/sda \
    --ignition-url https://example.com/example.ign
    ~~~
После завершения установки перезагрузите систему, используя `sudo reboot`. После завершения перезагрузки запустится первый процесс установки. В это время Ignition скопирует файлы конфигурации и запускает систему в соответсвии с требованиями.

Для более продвинутого уровня установки ISO, включая автоматизацию, читайте ниже. Для более подробной информации про образ live ISO, обратитесь к [ссылке на live image](https://docs.fedoraproject.org/en-US/fedora-coreos/live-reference/).
> **Совет**
> 
> Проверьте `coreos-installer install --help` для дополнительных возможностей по установке Fedora CoreOS.
>

# Установка из сети
***

> **Примечание** 
>  
> Загрузка образа live PXE требует не менее 2 Gb оперативной памяти с аргументом ядра `coreos.live.rootfs_url`, в ином случае 4 Gb. Вы также можете установить его на легаси  BIOS или режим UEFI, независимо от режима ОС, используемого после установки.
>
## Установка из PXE
Выполните следующие действия для установки образа из PXE:

* Загрузите ядро FCOS PXE, initramfs и образ rootfs:
 ```
 podman run --security-opt label=disable --pull=always --rm -v .:/data -w /data \
  quay.io/coreos/coreos-installer:release download -f pxe
  ```
  * Следуйте примеру `pxelinux.cfg` для загрузки установки обзазов с PXELINUX:
```
DEFAULT pxeboot
TIMEOUT 20
PROMPT 0
LABEL pxeboot
    KERNEL fedora-coreos-40.20240504.3.0-live-kernel-x86_64
    APPEND initrd=fedora-coreos-40.20240504.3.0-live-initramfs.x86_64.img,fedora-coreos-40.20240504.3.0-live-rootfs.x86_64.img coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=http://192.168.1.101:8000/config.ign
IPAPPEND 2
```
Более подробная информация об использовании этих данных содержится в [статье](https://dustymabe.com/2019/01/04/easy-pxe-boot-testing-with-only-http-using-ipxe-and-libvirt/) про тестирование установки PXE при помощи локальной ВМ и `libvirt`. Про остальные опции, поддерживающиеся командной строкой ядра, ознакомьтесть с [coreos-installer docs](https://coreos.github.io/coreos-installer/getting-started/#kernel-command-line-options-for-coreos-installer-running-as-a-service), но обратите внимание, что `coreos-installer pxe customize` (см. выше) отличается большей гибкостью. Для более полной информации об образе PXE ознакомьтесь с [сылками на live image](https://docs.fedoraproject.org/en-US/fedora-coreos/live-reference/).

## Установка из iPXE
 Устройству, поддерживающему iPXE, потребуется пердоставить актуальные Boot Script для извлечения и загрузки артефактов FCOS. 
 
 Следующий пример демонстрирует как загрузить их напрямую из инфраструктуры Fedora. По соображениям эффективности и надежности рекомендуется отразить их в локальной инфраструктуре, чтобы в дальнейшем как положено настроить `BASEURL`.
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
Про остальные опции, поддерживающиеся командной строкой ядра, ознакомьтесть с [coreos-installer docs](https://coreos.github.io/coreos-installer/getting-started/#kernel-command-line-options-for-coreos-installer-running-as-a-service), но обратите внимание, что `coreos-installer pxe customize` (см. выше) отличается большей гибкостью. Для более полной информации об образе PXE ознакомьтесь с [сылками на live image](https://docs.fedoraproject.org/en-US/fedora-coreos/live-reference/).

# Установка из контейнера
***
Используйте [контейнер](https://quay.io/repository/coreos/coreos-installer) `coreos-installer` из существующей системы для установки на подлключенное блок-устройство. К примеру (замените `docker` для `podman`, если потребуется):
```
sudo podman run --pull=always --privileged --rm \
    -v /dev:/dev -v /run/udev:/run/udev -v .:/data -w /data \
    quay.io/coreos/coreos-installer:release \
    install /dev/vdb -i config.ign
```
В данном примере `coreos-installer` загрузит последний стабильный FCOS metal image и установит его в `/dev/vdb`. Затем запустит файл Ignition `config.ign` текущей директории в образ. Используйте `--help` для просмотра всех доступных опций.