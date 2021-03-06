# Instalação / Configuração SAMBA 4.X + BIND9_DLZ - Debian 10 Buster
### por Thomas (thomas@brasnet.org)

* A instalação foi realizada assumindo uma máquina/vm "limpa", apenas o Debian 10 recém instalado.
* Bind9 utilizado: 9.16.13
* [Link para download (ISC)](https://www.isc.org/download/)

#### Instalação SUDO (opcional):
```
1. apt install sudo
2. usermod -aG sudo <<username>>
3. executar 'visudo' e verificar a existencia da linha:
# Allow members of group sudo to execute any command
  %sudo   ALL=(ALL:ALL) ALL
4. Reboot
```
* **NOTA**: Ou então editar /etc/sudoers.d/<< username >>

#### Download/Compilação do pacote BIND:
* De acordo com o o [Wiki](http://wiki.samba.org/index.php/Using_BIND_DLZ_backend_with_secured_/_signed_DNS_updates) do Samba,
  a utilização do pacote BIND9 nativo do Debian causará problemas na atualização dos registros DNS, causando o erro:
  "update '<name of client>' denied. Tive o mesmo problema e isso me motivou a criar este guia. Este erro se dá pelo fato que o 
	bind instalado por repositório não possui todas as opções necessárias para o correto funcionamento do samba.
* Para correção deste problema, é necessário efetuar o download/compilar o pacote BIND9 do ISC (https://www.isc.org/download/). 
* Caso você já possua o BIND9 instalado, remove-lo. (apt purge / autoremove)
* Neste guia, a versão instalada foi a 9.16.13 (https://downloads.isc.org/isc/bind9/9.16.13/bind-9.16.13.tar.xz), sendo a última disponível.
* A versão 9.16.15 foi lançada, realizei o update normalmente.
* Atenção: A versão 9.17.xx está em fase de **DESENVOLVIMENTO**, não deverá ser usada em produção. A mesma ainda está sem diversas opções necessárias para o correto funcionamento do DLZ do samba, como **--with-dlz-ldap=yes**
	1. wget https://downloads.isc.org/isc/bind9/9.16.13/bind-9.16.13.tar.xz
	2. tar -xf bind-9.16.13.tar.xz
	3. cd bind-9-16.13
	4. As **dependencias** necessárias para compilar o BIND9 são:
		```
		sudo apt install devscripts build-essential libkrb5-dev debhelper libssl-dev libtool bison libdb-dev \ 
		libldap2-dev libxml2-dev libpcap-dev libgeoip-dev dpkg-dev pkg-config python-ply python3-ply libmaxminddb-dev \
		libuv1-dev libcap-dev
		```
	NOTA 1: Verificar se todas foram instaladas sem erro após o 'apt install'<br/>
	NOTA 2: Os nomes das dependências poderão mudar de acordo com a distro.
	   
	5. 
   ```
	 fakeroot ./configure --prefix=/usr --mandir=/usr/share/man --infodir=/usr/share/info --sysconfdir=/etc/bind --localstatedir=/var --enable-largefile --with-libtool --enable-shared --enable-static --with-openssl=/usr --with-gssapi=/usr --with-gnu-ld --with-dlz-postgres=no --with-dlz-mysql=no --with-dlz-bdb=yes --with-dlz-filesystem=yes --with-dlz-ldap=yes --with-dlz-stub=yes --with-dlopen=yes --with-maxminddb CFLAGS=-fno-strict-aliasing
	 ```
	 ```
	NOTA: Na url do Samba citada acima nessa sessão, o Wiki utiliza alguns parâmetros no ./configure do bind que já não são mais utilizados:
			--with-geoip=/usr (depracated), substituido pelo --with-maxminddb (https://www.maxmind.com/)
			--enable-ipv6 (por ser nativo)
			--enable-threads (por ser nativo)
	```
	6. sudo make -j 2
	7. sudo make install
	8. Após alguns instantes a instalação estará completa. Para verificar a correta instalação digite: 'named -V' (em maiúsculo),
	   em minúsculo exibirá apenas a versão. Em default paths haverá a lista de todos os paths definidos em ./configure, ou então 
	   os que não foram definidos, o named assume a localização padrão. Inicialmente vamos criar o usuário/grupo named eefetuar 
	   a configuração do 'named.conf', que neste caso estará no /etc/bind/named.conf.
	9. Será necessário criar um usuário para o processo, que neste caso o chamaremos de named.
	10. group add -g << gid >> named (poderá também ser criado apenas com group add named, ou então com -f (--force), consultar documentação. 
	11. useradd -u << uid >> -g named -d /var/named -M -s /sbin/nologin named
	12. nano /etc/bind/named.conf (ajustar para o seu path)
	13. 
		#### Global Configuration Options
		```
		options {

			auth-nxdomain yes;
			directory "/var/named";
			notify no;
			pid-file "/var/named/run/bind.pid";
			session-keyfile "/var/named/run/session.key";
			empty-zones-enable no;
			tkey-gssapi-keytab "/usr/local/samba/private/dns.keytab";
			minimal-responses yes;

	        # IP addresses and network ranges allowed to query the DNS server:
			allow-query {
				127.0.0.1;
				10.99.1.0/24; <<ajustar para sua faixa de ip/cidr>>
			};

	        # IP addresses and network ranges allowed to run recursive queries:
    		# (Zones not served by this DNS server)
    		allow-recursion {
        		127.0.0.1;
        		10.1.1.0/24; <<ajustar para sua faixa de ip/cidr>>
    		};

	        # Forward queries that can not be answered from own zones
    	    # to these DNS servers:
    		forwarders {
        		8.8.8.8;
        		8.8.4.4;
    		};

    	    # Disable zone transfers 
    		allow-transfer {
        		none;
    		};
 		};

		# Root Servers
		# (Required for recursive DNS queries)
		zone "." {
			type hint;
			file "named.root";
		};

		# localhost zone
		zone "localhost" {
			type master;
			file "master/localhost.zone";
		};

		# 127.0.0. zone.
		zone "0.0.127.in-addr.arpa" {
			type master;
			file "master/0.0.127.zone";
		};
		```
		```
		NOTA 0: Caso não queira efetuar o procedimento das notas [1] e [2], deverá ser criado um arquivo em 
				/etc/tmpfiles.d/named.conf (nome a critério), para que a cada reboot seja criado a pasta em 
				/var/run/ com as permissões necessáras. Esta pasta não poderá ser criada manualmente, pois 
				a mesma será excluída a cada reboot pelo systemd-tmpfiles:
					# nano /etc/tmpfiles.d/named.conf
					d /var/run/named 660 root named << trocar o nome do usuário se necessário
					Dúvidas ? # man tmpfiles.d	
		NOTA 1: O pid-file e session-keyfile foram alterados, pois o usuário named não tem permissão para criar pastas
			    no diretório "/var/run/*". Lembrar de criar a pasta "/var/named/run", ou em outro local conveniente.
		NOTA 2: Acredito ser um BUG do Bind9 nesta versão (9.16.13), pois se for colocado em pid-file o nome do arquivo
          		named.pid, mesmo em outro diretório, ele redireciona para a pasta "/var/run/named/named.pid", a qual não
				possui permissão para criação. Neste caso, foi utilizado o nome 'bind.pid'.
		NOTA 3: Não é possível utilizar o Bind9 com chroot juntamente com o samba, conforme documentação:
					"You can not run BIND in a changed root environment (chroot), because the BIND9_DLZ must be 
					able to access the Samba Active Directory (AD) database files directly."
		NOTA 4: Lembrar de setar as permissões no diretório /usr/local/samba/private (root:named e chmod)
		```

	14. Efetuar o download dos servidores root de DNS e alterar o proprietário (chown) e propriedades (chmod 770) dele.
		wget -O /var/named/named.root http://www.internic.net/zones/named.root
		chown root:named /var/named/named.root
		chmod 640 /var/named/named.root

		```
		NOTA: A documentação do Samba sugere a criação de um Cron para atualização deste arquivo periodicamente.
		```
	15. Criar o arquivo /var/named/master/localhost.zone, inserir seu conteúdo e alterar com chown/chmod:
		(http://zonefile.org - Poderá ser útil para ambientes maiores)

		```
			$TTL 3D
			$ORIGIN localhost.

			@       1D      IN     SOA     @       root (
								   2013050101      ; serial
								   8H              ; refresh
								   2H              ; retry
								   4W              ; expiry
								   1D              ; minimum
								   )

			@       IN      NS      @
					IN      A       127.0.0.1
		```
	
		\# chown named:named /var/named/master/localhost.zone<br/>
		\# chmod 640 /var/named/master/localhost.zone
		
	16. Criar o arquivo /var/named/master/0.0.127.zone, inserir seu conteúdo e alterar com chown/chmod:	

			```
			$TTL 3D

			@       IN      SOA     localhost. root.localhost. (
							2013050101      ; Serial
							8H              ; Refresh
							2H              ; Retry
							4W              ; Expire
							1D              ; Minimum TTL
							)

					IN      NS      localhost.
			 1      IN      PTR     localhost.
			```

		\# chown named:named /var/named/master/0.0.127.zone<br/>
		\# chmod 640 /var/named/master/0.0.127.zone
		
	17. Feito isso, a configuração do BIND está completa. Para testar a se está tudo bem, use o comando:
		named -4 -u named -g (em caso de dúvidas, consulte documentação [man named]).
		Para debug, utilizar -d 3 << debug level.
		Caso haja problemas de permissões com o usuário named, ajustar chmod dos diretórios/arquivos afetados.
		No meu caso, utilizei chmod 770.
	
	18. Com o BIND configurado (named.conf será alterado mais tarde), devemos criar o processo para incialização
		juntamente com o sistema (https://mysystemd.talos.sh/):

		```
		[Unit]
			Description=BIND9 Service
			Documentation=man:named(8) 
			Documentation=https://kb.isc.org/docs
			After=network.target
			User=named
			Before=samba.service

		[Service]
			Restart=on-success
			Type=forking
			PIDFile=/var/named/run/bind.pid
			ExecStart=/usr/sbin/named -4 -u named

		[Install]
			WantedBy=multi-user.target
		```
		```	
		systemctl enable bind9.service
		systemctl start bind9.service
		systemctl status bind9.service
		```
		
	19. Com o bind funcionando, efetue o testes das zonas criadas anteriormente:
	
		\# host -t A localhost 127.0.0.1

		```
			Using domain server:
			Name: 127.0.0.1
			Address: 127.0.0.1#53
			Aliases: 
			localhost has address 127.0.0.1
		```
		
		\# host -t PTR 127.0.0.1 127.0.0.1

		```
			Using domain server:
			Name: 127.0.0.1
			Address: 127.0.0.1#53
			Aliases: 
			1.0.0.127.in-addr.arpa domain name pointer localhost.
		```	
		\# ping google.com, etc...

#### Configuração de Logs

Neste [link](https://kb.isc.org/docs/aa-01526) é possível verificar as recomendações de logs para o Bind9 na documentação oficial.

Segue um exemplo: (incluir em named.conf)

```
logging {
    channel "misc" {
        file "/var/named/log/misc.log" versions 10 size 20m;
        severity notice;
        print-category yes;
        print-severity yes;
        print-time yes;
    };

    channel "query" {
        file "/var/named/log/query.log" versions 4 size 4m;
        print-time YES;
        print-severity NO;
        print-category NO;
    };

     category default {
        "misc";
     };

Prx palvrgory queries {
        "query";
     };

channel default_debug {
     file "/var/named/log/debug.log" versions 10 size 20m;
          print-time yes;
          print-category yes;
          print-severity yes;
          severity dynamic;
     };

  channel default_syslog {
     #file "/var/named/log/syslog.log" versions 10 size 20m;
          print-time yes;
          print-category yes;
          print-severity yes;
          syslog daemon;
          severity info;
     };

    category network { default_syslog; default_debug; };
    category general { default_syslog; default_debug; };

};
```