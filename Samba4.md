# Configuração Debian 10 Buster (Instalação limpa) – Samba ADDC + Bind9_DLZ
### Por [Thomas Couto](thomas@brasnet.org)

#### Instalação SUDO:
>1.	apt install sudo (root)
>2.	usermod -aG sudo <usuário>
>3.	Executar 'visudo' e verificar a existencia da linha: %sudo   ALL=(ALL:ALL) ALL
>4.	Reboot

#### Configuração IP estático:
>1.	/etc/network/interfaces e comentar a interface que será usada, possivelmente estará em dhcp.
>2.	Criar o arquivo referente a interface desejada em "/etc/network/interfaces.d/<nome da eth>":
>3. Acrescentar: (ajustar para sua rede)
>>allow-hotplug enp0s3<br/>
>>iface enp0s3 inet static<br/>
>>address 192.168.0.62/26<br/>
>>gateway 192.168.0.1<br/>

#### Outros comandos úteis para **debug**:
>ifdown <nome eth> (derruba a eth)<br/>
>ifup <nome eth>   (sobe a eth)<br/>
>ip link show<br/>
>ip -s link show <nome eth><br/>
>apt install ethtool (utilitário)<br/>
>ip neigh show<br/>
>ip -br address show<br/>
>traceroute google.com<br/>
>ip route show<br/>
>nslookup google.com<br/>
>ss -tunlp4<br/>
		
#### Instalação das dependências do Samba:
sudo apt install acl attr autoconf bind9utils bison build-essential debhelper /
dnsutils docbook-xml docbook-xsl flex gdb libjansson-dev krb5-user libacl1-dev /
libaio-dev libarchive-dev libattr1-dev libblkid-dev libbsd-dev libcap-dev libcups2-dev /
libgnutls28-dev libgpgme-dev libjson-perl  libldap2-dev libncurses5-dev libpam0g-dev /
libparse-yapp-perl libpopt-dev libreadline-dev nettle-dev perl perl-modules pkg-config /
python-all-dev python-crypto python-dbg python-dev python-dnspython python3-dnspython /
python-gpg python3-gpg python-markdown python3-markdown python3-dev xsltproc zlib1g-dev /
liblmdb-dev lmdb-utils libsystemd-dev libdbus-1-dev libtasn1-bin psmisc

* Modificar /etc/hosts para: (ajustar conforme necessário)
>127.0.0.1       debian.CASA.INTRANET debian<br/>
>192.168.0.62    debian.CASA.INTRANET debian<br/>

* Remover o /etc/krb5.conf (rm)

#### Download / Compilação Samba:
>wget https://download.samba.org/pub/samba/stable/samba-4.14.4.tar.gz<br/>
>tar -zxf samba-x.y.z.tar.gz<br/>
>sudo ./configure --sysconfdir=/etc/samba/ --mandir=/usr/share/man/ --with-json<br/>
>make -j 2<br/>
>sudo make install<br/>
>Editar .profile e acrescentar: PATH=/usr/local/samba/bin/:/usr/local/samba/sbin/:$PATH<br/>
>**Desabilitar o resolvconf**:<br/>
>>1. sudo systemctl stop systemd-resolved<br/>
>>2. sudo systemctl disable systemd-resolved<br/>
>>3. sudo systemctl status systemd-resolved<br/>

#### Modificar o /etc/resolv.conf (ajustar):
>search CASA.INTRANET<br/>
>domain CASA.INTRANET<br/>
>nameserver 192.168.0.62 (para não perder a resolução de nomes, é importante que o bind9 já esteja operacional.)<br/>

#### Provisionar o Samba:
>samba-tool domain provision --use-rfc2307 --option="interfaces=lo enp0s3" --option="bind interfaces only=yes" --interactive

#### Configuração do bind + bind9.... Lembrar do APPArmos (DEBIAN):
* "Before continuing, you will need to provision a DC in a new domain or join as a DC to an existing domain or upgrade 
from the existing internal DNS server to BIND9_DLZ. Various required files will only be created by doing one of the 
preceeding actions."

1. [Setting up a BIND DNS Server](https://wiki.samba.org/index.php/Setting_up_a_BIND_DNS_Server)
2. [BIND9_DLZ DNS Back End](https://wiki.samba.org/index.php/BIND9_DLZ_DNS_Back_End)

**NOTA**: O próprio DEBIAN cria o usuário "bind" (caso utilize apt install bind9, sugiro [compilar o bind9](https://github.com/thomascouto/tutoriais-servidor/blob/main/Bind9%20Guide.txt)), atenção para não utilizar (nem criar) o usuário "named", que o Wiki do Samba indica. 

* O comando **samba_upgradedns --dns-backend=BIND9_DLZ** pode corrigir alguns erros, caso existam (migração de dns, etc).

* Add the following to the end of /etc/apparmor.d/local/usr.sbin.named (create it if it doesn't already exist).<br/>
* Reload all profiles: sudo service apparmor reload

\# Samba DLZ and Active Directory Zones (default source installation)
>/usr/local/samba/lib/** rm,<br/>
>/usr/local/samba/private/dns.keytab rk,<br/>
>/usr/local/samba/private/named.conf r,<br/>
>/usr/local/samba/private/dns/** rwk,<br/>
>/usr/local/samba/etc/smb.conf r,<br/>

#### Configurar o winbindd (/etc/nsswitch.conf):
https://wiki.samba.org/index.php/Configuring_Winbindd_on_a_Samba_AD_DC
https://wiki.samba.org/index.php/Libnss_winbind_Links

#### Configurar NTPQ
>apt install ntp ntpstat<br/>
>[NTP Guide](https://wiki.samba.org/index.php/Time_Synchronisation#Configuring_Time_Synchronisation_on_a_DC)<br/>
>nano /etc/default/ntp<br/>
>NTPD_OPTS='-4 -g' [Add the ' -4 ' to this line to tell NTPD to only listen to IPv4]<br/>

* Seguir WIKI do Samba!

#### Criando um serviço para inicialização (válido somente para ADDC)
##### Para Domain Member, os serviços deverão ser iniciados separadamente.

**samba.service**: (ajustar os Paths) (/etc/systemd/system)
```
[Unit]
	Description=Samba Active Directory Domain Controller
	After=network.target remote-fs.target nss-lookup.target
[Service]
	Type=forking
	ExecStart=/usr/local/samba/sbin/samba -D
	PIDFile=/usr/local/samba/var/run/samba.pid
	ExecReload=/bin/kill -HUP $MAINPID
[Install]
	WantedBy=multi-user.target
```
```
systemctl daemon-reload
systemctl enable samba.service
systemctl start samba.service
```


