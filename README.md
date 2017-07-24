# Recuperar senha da camera VIP 1120

Estou escrevendo esse documento pra compartilhar um pouco das minhas aventuras com a camera Intelbras VIP 1120 e um pouco da minha frustração ao tentar trocar a senha do `admin` e depois descobrir que a nova senha não funcionava.

Depois de pesquisar bastante encontrei esse video no youtube https://www.youtube.com/watch?v=BWY2SmiORis ensinando a resetar a camera VIP S3230, abri minha camera e descobri que ela não tinha o tal jumper pra resetar. Então resolvi procurar pelo código do chip que esta na placa, Hi3018, e descobri que se trata de um chip genérico, que muitas cameras da china utiliza.

![Hi3518.jpg](/images/Hi3518.jpg)

Ao vermos a imagem acima, percebemos que a camera possui uma interface serial, então resolve conectar essa interface no meu notebook, pra isso usei um adaptador serial-usb, no meu caso foi um Aruidno Uno mesmo. Conectei na porta serial usando o comando, no caso aqui estou usando um Mac:

```bash
sudo cu --speed 115200 --line /dev/cu.usbmodem14621
```

liguei minha camera, e o resultado foi:

```bash
U-Boot 2010.06-svn (Oct 14 2015 - 15:07:23)

DRAM:  256 MiB
Check spi flash controller v350... Found
Spi(cs1) ID: 0xC2 0x20 0x17 0xC2 0x20 0x17
Spi(cs1): Block:64KB Chip:8MB Name:"MX25L6406E"
envcrc 0x288fc0bb
ENV_SIZE = 0xfffc
In:    serial
Out:   serial
Err:   serial
Press Ctrl+C to stop autoboot
CFG_BOOT_ADDR:0x58040000
8192 KiB hi_sfc at 0:0 is now current device

### boot load complete: 1973968 bytes loaded to 0x82000000
### SAVE TO 80008000 !
## Booting kernel from Legacy Image at 82000000 ...
   Image Name:   linux
   Image Type:   ARM Linux Kernel Image (uncompressed)
   Data Size:    1973904 Bytes = 1.9 MiB
   Load Address: 80008000
   Entry Point:  80008000


load=0x80008000,_bss_end=80829828,image_end=801e9e90,boot_sp=807c7168
   Loading Kernel Image ... OK
OK

Starting kernel ...

Uncompressing Linux... done, booting the kernel.
```

Nada de muito util, mas consegui logar diretamente na camera, e se você notar apareceu bem rapidamente a mensagem: `Press Ctrl+C to stop autoboot`, fazendo isso o boot da camera é interrompido e você é jogado num console onde pode executar vários comandos, entre eles, ler/gravar na flash, fazer download/uploads via tftp, e é aqui que a coisa fica divertida.

# Aviso

Não me responsabilizo por qualquer dano causado a sua camera ao executar os passos listados aqui.

É necessário ter conhecimentos intermediários ou avançados de Linux, se ao começar a ler esse documento você não entender o que se passa é melhor levar sua camera a assistencia da Intelbras.

# Material necessário

* Linux (no meu caso usei Virtualbox)
* Adaptador serial-usb (Arduino Uno serve)

# Preparando

Antes de começar é necessário ter um servidor tfpd rodando na sua rede, pra isso no seu linux execute:

```bash
sudo apt-get install tftpd-hpa
```

Você talvez precise trocar as permissões do diretorio `/var/lib/tftpboot` para que o usuário tfpd possa ler/escrever nesse diretório

```bash
sudo chown tftp:nogroup /var/lib/tftpboot/
sudo chmod a+w /var/lib/tftpboot/
sudo service tftpd-hpa restart
```

# Backup

Feito isso, vamos começar efetuando um backup da rom da camera, caso algo de errado ainda é possivel restaurar o software original. Pra isso vá ao console da camera e digite

```bash
printenv
```

Você deverar ver algo parecido com isso:

```bash
hisilicon # printenv
bootcmd=setenv setargs setenv bootargs ${bootargs};run setargs;fload;bootm 0x82000000
bootdelay=1
baudrate=115200
bootfile="uImage"
da=mw.b 0x82000000 ff 1000000;tftp 0x82000000 u-boot.bin.img;sf probe 0;flwrite
du=mw.b 0x82000000 ff 1000000;tftp 0x82000000 user-x.cramfs.img;sf probe 0;flwrite
dr=mw.b 0x82000000 ff 1000000;tftp 0x82000000 romfs-x.cramfs.img;sf probe 0;flwrite
dw=mw.b 0x82000000 ff 1000000;tftp 0x82000000 web-x.cramfs.img;sf probe 0;flwrite
dc=mw.b 0x82000000 ff 1000000;tftp 0x82000000 custom-x.cramfs.img;sf probe 0;flwrite
up=mw.b 0x82000000 ff 1000000;tftp 0x82000000 update.img;sf probe 0;flwrite
ua=mw.b 0x82000000 ff 1000000;tftp 0x82000000 upall_verify.img;sf probe 0;flwrite
tk=mw.b 0x82000000 ff 1000000;tftp 0x82000000 uImage; bootm 0x82000000
dd=mw.b 0x82000000 ff 1000000;tftp 0x82000000 mtd-x.jffs2.img;sf probe 0;flwrite
ipaddr=192.168.1.10
serverip=192.168.1.107
netmask=255.255.255.0
bootargs=mem=${osmem} console=ttyAMA0,115200 root=/dev/mtdblock1 rootfstype=cramfs mtdparts=hi_sfc:256K(boot),3520K(romfs),2560K(user),1280K(web),256K(custom),320K(mtd)
ethaddr=58:10:8c:30:ba:4c
HWID=8043420004048425
NID=0x0005
osmem=44M
appSystemLanguage=Portugal
appVideoStandard=PAL
stdin=serial
stdout=serial
stderr=serial
verify=n
ver=U-Boot 2010.06-svn (Oct 14 2015 - 15:07:23)
```

Olhando a string `mtdparts=hi_sfc:256K(boot),3520K(romfs),2560K(user),1280K(web),256K(custom),320K(mtd)` eu inferi que a flash da camera estava dividida em 6 partiçoes:

| Partição | Nome   | Offset   | Length   |
|----------|--------|----------|----------|
| 0        | boot   | 0x0      | 0x40000  |
| 1        | romfs  | 0x40000  | 0x370000 |
| 2        | user   | 0x3b0000 | 0x280000 |
| 3        | web    | 0x630000 | 0x140000 |
| 4        | custom | 0x770000 | 0x40000  |
| 5        | mtd    | 0x7b0000 | 0x50000  |

Antes porém vamos configurar o endereço do nosso servidor tfpd pra enviar os arquivos:

```bash
setenv ipaddr XXX.YYY.ZZZ.100
setenv gatewayip XXX.YYY.ZZZ.253
setenv serverip XXX.YYY.ZZZ.2
```

onde `ipaddr` será o ip da camera na rede, escolha um ip que não vá colidir com ip de outros dispositivos na sua rede, `gatewayip` é o endereço do seu router e `serverip` é o endereço do seu servidor tfpd

Então para salvar as partições da flash no servidor execute

```bash
sf probe 0
```

para ativar a flash

```bash
sf read 0x82000000 0x0 0x40000
tftp 0x82000000 u-boot.bin.img 0x40000
```

```bash
sf read 0x82000000 0x40000 0x370000
tftp 0x82000000 romfs-x.cramfs.img 0x370000
```

```bash
sf read 0x82000000 0x3b0000 0x280000
tftp 0x82000000 user-x.cramfs.img 0x280000
```

```bash
sf read 0x82000000 0x630000 0x140000
tftp 0x82000000 web-x.cramfs.img 0x140000
```

```bash
sf read 0x82000000 0x770000 0x40000
tftp 0x82000000 custom-x.cramfs.img 0x40000
```

```bash
sf read 0x82000000 0x7b0000 0x50000
tftp 0x82000000 mtd-x.jffs2.img.img 0x50000
```

Depois disso você deverá ver no seu servidor tftpd os arquivos

```bash
ls -l /var/lib/tftpboot/

-rw-rw-rw- 1 tftp tftp  262144 Jul 21 19:54 custom-x.cramfs.img
-rw-rw-rw- 1 tftp tftp  327680 Jul 22 21:47 mtd-x.jffs2.img
-rw-rw-rw- 1 tftp tftp 3604480 Jul 21 19:51 romfs-x.cramfs.img
-rw-rw-rw- 1 tftp tftp  262144 Jul 21 19:51 u-boot.bin.img
-rw-rw-rw- 1 tftp tftp 2621440 Jul 21 19:52 user-x.cramfs.img
-rw-rw-rw- 1 tftp tftp 1310720 Jul 21 19:54 web-x.cramfs.img
```

Salve esses arquivos em um lugar seguro pra poder recuperá-los depois.

# Editando a senha

Se você possui conhecimentos avançados de linux você perceberá que esse arquivos são imagens de sistemas de arquivos do tipo cramfs, squashfs e jffs2. Você poderá montá-los usando o comando mount e pode até modificá-los e gravá-los de volta na flash da camera. Foi assim que eu recuperei a senha do usuário admin, mountei o arquivo `mtd-x.jffs2.img` usando o script `jffs2_mount_loop0.sh`, modifiquei o arquivo `Config/Account1` e gravei a imagem de volta na flash da camera. Como a senha fica criptogravada nesse arquivo, eu tinha que saber como seria a senha criptografada pra editar esse arquivo, por algum motivo desconhecido, existe o arquivo `Config/Account2` que ao que parece é a versão desse arquivo com a senha antiga, então eu copiei a senha que estava em um para o outro, no caso a senha era `6QNMIQGe`, que é a versão criptografada de `admin`. Para modificar as images você precisará instalar o `mtd-utils`:

```bash
sudo apt-get install mtd-utils
```

Se você não estiver muito afim de fazer isso, eu já disponibilizei o arquivo no github já alterado, com o usuário `admin` senha `admin`.

# Gravando

Depois de editado o arquivo `mtd-x.jffs2.img` vamos fazer o download para a camera:

```bash
sf probe 0
tftp 0x82000000 mtd-x.jffs2.img
sf erase 0x7b0000 0x50000
sf write 0x82000000 0x7b0000 0x50000
```

Esses comandos ira baixar o arquivo `mtd-x.jffs2.img` pra memória da camera no endereço `0x82000000`, depois ira apagar a partição 5 (preste bem atenção nos valores digitados), depois ira gravar o conteudo da memória `0x82000000` na flash.

# Comandos

```bash
sf read <address> <offset> <length>
sf write <address> <offset> <length>
sf erase <offset> <length>
tftp <address> <filename> [<length>]
```


