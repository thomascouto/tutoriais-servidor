# Configuração Debian 10 Buster (Instalação limpa) – Samba ADDC + Bind9_DLZ
### Por Thomas Couto - thomas@brasnet.org

## Instalação SUDO:
> 1.	apt install sudo (root)
> 2.	usermod -aG sudo <usuário>
> 3.	Executar 'visudo' e verificar a existencia da linha:
> %sudo   ALL=(ALL:ALL) ALL
> 4.	Reboot

## Configuração IP estático:
> 1.	/etc/network/interfaces e comentar "# The primary network interface"
> 2.	Criar o arquivo referente a interface em "/etc/network/interfaces.d/<nome da eth>":

\# The primary network interface (AJUSTAR CONFORME NECESSÁRIO):
> allow-hotplug enp0s3
> iface enp0s3 inet static
> address 192.168.0.62/26
> gateway 192.168.0.1

* Outros comandos úteis para debug:
> ifdown <nome eth> (derruba a eth)<br/>
> ifup <nome eth>   (sobe a eth)<br/>
> ip link show<br/>
> ip -s link show <nome eth><br/>
> apt install ethtool (utilitário)<br/>
> ip neigh show<br/>
> ip -br address show<br/>
> traceroute google.com<br/>
> ip route show<br/>
> nslookup google.com<br/>
> ss -tunlp4<br/>
		
## Instalação das dependências do Samba:
sudo apt install acl attr autoconf bind9utils bison build-essential debhelper /
dnsutils docbook-xml docbook-xsl flex gdb libjansson-dev krb5-user libacl1-dev /
libaio-dev libarchive-dev libattr1-dev libblkid-dev libbsd-dev libcap-dev libcups2-dev /
libgnutls28-dev libgpgme-dev libjson-perl  libldap2-dev libncurses5-dev libpam0g-dev /
libparse-yapp-perl libpopt-dev libreadline-dev nettle-dev perl perl-modules pkg-config /
python-all-dev python-crypto python-dbg python-dev python-dnspython python3-dnspython /
python-gpg python3-gpg python-markdown python3-markdown python3-dev xsltproc zlib1g-dev /
liblmdb-dev lmdb-utils libsystemd-dev libdbus-1-dev libtasn1-bin psmisc

\# Modificar /etc/hosts para: (ajustar conforme necessário)
127.0.0.1       debian.CASA.INTRANET debian
192.168.0.62    debian.CASA.INTRANET debian

* Remover o /etc/krb5.conf (rm)

## Download / Compilação Samba:
> wget samba.org (https://download.samba.org/pub/samba/stable/)
> tar -zxf samba-x.y.z.tar.gz
> sudo ./configure --sysconfdir=/etc/samba/ --mandir=/usr/share/man/ --with-json
> make -j 2
> sudo make install
> Editar .profile e acrescentar: PATH=/usr/local/samba/bin/:/usr/local/samba/sbin/:$PATH
> \# Desabilitar o resolvconf:
>>	1. sudo systemctl stop systemd-resolved
>>	2. sudo systemctl disable systemd-resolved
>>	3. sudo systemctl status systemd-resolved

- Modificar o /etc/resolv.conf (ajustar):
search CASA.INTRANET
domain CASA.INTRANET
nameserver 192.168.0.62 (para não perder a resolução de nomes, é importante que o bind9 já esteja operacional.)

>>> Provisionar o Samba:
samba-tool domain provision --use-rfc2307 --option="interfaces=lo enp0s3" --option="bind interfaces only=yes" --interactive

>>> Configuração do bind + bind9.... Lembrar do APPArmos (DEBIAN):
"Before continuing, you will need to provision a DC in a new domain or join as a DC to an existing domain or upgrade 
from the existing internal DNS server to BIND9_DLZ. Various required files will only be created by doing one of the 
preceeding actions."

https://wiki.samba.org/index.php/Setting_up_a_BIND_DNS_Server
https://wiki.samba.org/index.php/BIND9_DLZ_DNS_Back_End

NOTA: O próprio DEBIAN cria o usuário "bind" (caso utilize apt install bind9, sugiro compilar o bind9), cuidado para não 
utilizar (nem criar) o usuário "named", que o Wiki do Samba indica. 

samba_upgradedns --dns-backend=BIND9_DLZ -> CORRIGE ALGUNS ERROS !

Add the following to the end of /etc/apparmor.d/local/usr.sbin.named (create it if it doesn't already exist).
Reload all profiles: sudo service apparmor reload

# Samba DLZ and Active Directory Zones (default source installation)
/usr/local/samba/lib/** rm,
/usr/local/samba/private/dns.keytab rk,
/usr/local/samba/private/named.conf r,
/usr/local/samba/private/dns/** rwk,
/usr/local/samba/etc/smb.conf r,

>>> Configurar o winbindd (/etc/nsswitch.conf):
https://wiki.samba.org/index.php/Configuring_Winbindd_on_a_Samba_AD_DC
https://wiki.samba.org/index.php/Libnss_winbind_Links

>>> Configurar NTPQ
apt install ntp ntpstat
https://wiki.samba.org/index.php/Time_Synchronisation#Configuring_Time_Synchronisation_on_a_DC
nano /etc/default/ntp
NTPD_OPTS='-4 -g' [Add the ' -4 ' to this line to tell NTPD to only listen to IPv4]

>>>  Seguir WIKI do Samba
samba.service: (ajustar os Paths) (/etc/systemd/system #nano samba.service)
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

- systemctl daemon-reload
- systemctl enable myservice.service
- systemctl start myservice.service


