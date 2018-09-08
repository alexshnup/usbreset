# usbreset

## Как программно перезагрузить зависшее USB устройство
##### Для резервирования интернет соединения я использую 3G модем Huawei E173, подключенный в USB порт маршрутизатора. Соединение через него всегда поднято в режиме горячего резерва (для переключения на модем достаточно сбросить default route с основного соединения). Но есть одна проблема - периодически модем "зависает" и соединение теряется.

Как правило достаточно перезапустить pppd, но вчера модем перестал реагировать совсем. В логе появились сообщения, которые относятся к проблемам скорее аппаратным.

```
xhci_hcd 0000:02:00.0: WARN Event TRB for slot 1 ep 4 with no TDs queued?
xhci_hcd 0000:02:00.0: WARN Event TRB for slot 1 ep 4 with no TDs queued?
xhci_hcd 0000:02:00.0: WARN Event TRB for slot 1 ep 4 with no TDs queued?
xhci_hcd 0000:02:00.0: ERROR Transfer event TRB DMA ptr not part of current TD
xhci_hcd 0000:02:00.0: ERROR Transfer event TRB DMA ptr not part of current TD
xhci_hcd 0000:02:00.0: ERROR Transfer event TRB DMA ptr not part of current TD
```

Перезапуск pppd ничего не дал, похоже модем не отвечает ни на одну команду.

```
Apr 24 10:02:11 inet chat[6761]: abort on (\nBUSY\r)
Apr 24 10:02:11 inet chat[6761]: abort on (\nERROR\r)
Apr 24 10:02:11 inet chat[6761]: abort on (\nNO ANSWER\r)
Apr 24 10:02:11 inet chat[6761]: abort on (\nNO CARRIER\r)
Apr 24 10:02:11 inet chat[6761]: abort on (\nNO DIALTONE\r)
Apr 24 10:02:11 inet chat[6761]: abort on (\nRINGING\r\n\r\nRINGING\r)
Apr 24 10:02:11 inet chat[6761]: send (^MAT^M)
Apr 24 10:02:11 inet chat[6761]: timeout set to 12 seconds
Apr 24 10:02:11 inet chat[6761]: expect (OK)
Apr 24 10:02:23 inet chat[6761]: alarm
Apr 24 10:02:23 inet chat[6761]: Failed
```

Попытки переинициализировать модем программно не увенчались успехом, поскольку его устройство (/dev/ttyUSB0) не отвечает на AT команды. Остается только отключить и снова включить модем в порт. Но сначала решил попробовать метод, на который наткнулся недавно в интернете.

Для сброса нужной шины USB нам потребуется скомпилировать бинарник. Чтобы не компилировать его каждый раз снова и пользоваться им на практически любой машине я буду компилировать его статически.

```
$ wget https://gist.githubusercontent.com/x2q/5124616/raw -O usbreset.c
$ gcc -Wall -static -o usbreset usbreset.c
$ sudo install -o root -g root -m 0755 usbreset /usr/local/sbin
$ lsusb | grep Huawei
Bus 001 Device 002: ID 12d1:1001 Huawei Technologies Co., Ltd. E169/E620/E800 HSDPA Modem
$ sudo usbreset /dev/bus/usb/001/002
Error in ioctl: No such device
```

Несмотря на ошибку в логе появились записи, свидетельствующие о "перезагрузке" модема.

```
$ dmesg | tail
usb 1-6: New USB device strings: Mfr=3, Product=2, SerialNumber=0
usb 1-6: Product: HUAWEI Mobile
usb 1-6: Manufacturer: HUAWEI Technology
usb 1-6: configuration #1 chosen from 1 choice
option 1-6:1.0: GSM modem (1-port) converter detected
usb 1-6: GSM modem (1-port) converter now attached to ttyUSB0
option 1-6:1.1: GSM modem (1-port) converter detected
usb 1-6: GSM modem (1-port) converter now attached to ttyUSB1
option 1-6:1.2: GSM modem (1-port) converter detected
usb 1-6: GSM modem (1-port) converter now attached to ttyUSB2
```

Попробуем подключиться к нему и выполнить несколько AT команд.

```
$ screen /dev/ttyUSB0 115200
ATZ
OK
ATI
Manufacturer: huawei
Model: E173
Revision: 11.126.16.00.715
IMEI: 861976004215827
+GCAP: +CGSM,+DS,+ES

OK
```

Модем ожил и теперь можно включать соединение.
