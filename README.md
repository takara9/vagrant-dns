# vagrant-dns

CentOS8でDNSサーバーを構築するVagrant + Ansible playbookです。
テストや開発のために、プライベートなDNSサーバーが欲しいと言った時に簡単に起動して、すぐに利用でます。


## 起動方法

起動方法は３ステップで起動します。
仮想サーバーのIPアドレスは、`172.2.1.250` が設定されているので Vagrantfile と
playbook/bind/templates/named.conf.j2 に設定されたアドレスを変更してください。

~~~
$ git clone https://github.com/takara9/vagrant-dns
$ cd vagrant-dns
$ vagrant up
~~~

## 設定ファイルの使用法

DNSの設定ファイルは、playbook/bind/vars/main.yml です。
このファイルを編集してDNSの登録を実施します。

~~~
.
├── README.md
├── Vagrantfile
├── ansible.cfg
└── playbook
    ├── bind
    │   ├── defaults
    │   │   └── main.yml
    │   ├── handlers
    │   │   └── main.yml
    │   ├── hosts
    │   ├── tasks
    │   │   └── main.yml
    │   ├── templates
    │   │   ├── db.forward.j2
    │   │   ├── db.reverse.j2
    │   │   ├── named.conf.j2
    │   │   ├── named.conf.local.j2
    │   │   └── resolv.j2
    │   └── vars
    │       └── main.yml
    └── install_bind.yml
~~~

このファイルを編集して、`vagrant up`によって、正引きと逆引きのzoneファイルを自動生成します。
そして、DNSサーバーまでを起動します。


設定ファイルの内容と項目の説明を以下に記述します。

~~~file:playbook/bind/vars/main.yml
---
hostname: server1

domain: mahot.org
reverse_domain: 1.20.172.in-addr.arpa.

dns_record:
 - name: www
   type: A
   ipaddress: 172.20.1.100
 - name: dev
   type: A
   ipaddress: 172.20.1.101
~~~

* hostname: 値をセットすると、Vagrantfileで指定したホスト名を上書
* domain: 実在しないドメイン名を設定
* reverse_domain: 172.20.1.X であれば、1.20.172.in-addr.arpaとして記述
* name: ホスト名
* type: A はホストネームであることを表すので固定
* ipaddress: IPアドレスをベタ書きする

上記の設定ファイルから、以下のように/etc/named以下にファイルを生成する

~~~
[root@server1 vagrant]# tree /etc/named
/etc/named
|-- named.conf.local
`-- zones
    |-- db.1.20.172.in-addr.arpa.
    `-- db.mahot.org

1 directory, 3 files
~~~


## 動作確認方法

`systemctl status named.service` を実行することで、正常に稼働しているか確認できる。

~~~
[root@server1 vagrant]# systemctl status named.service
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2020-01-06 13:23:50 UTC; 21min ago
  Process: 6231 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/S>
  Process: 6228 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin>
 Main PID: 6233 (named)
    Tasks: 4 (limit: 11525)
   Memory: 57.2M
   CGroup: /system.slice/named.service
           └─6233 /usr/sbin/named -u named -c /etc/named.conf

Jan 06 13:23:50 server1.takara9.org named[6233]: zone 1.20.172.in-addr.arpa/IN: loaded serial 7
Jan 06 13:23:50 server1.takara9.org named[6233]: all zones loaded
Jan 06 13:23:50 server1.takara9.org named[6233]: running
Jan 06 13:23:50 server1.takara9.org systemd[1]: Started Berkeley Internet Name Domain (DNS).
Jan 06 13:33:15 server1.takara9.org named[6233]: resolver priming query complete
~~~

## ドメインの正引き

出来ることを箇条書きにする。

* 独自のドメインと、インターネット上のドメインの両方を解決可能
* nslookupで仮想サーバーのIPアドレスを指定してIPアドレスを解決できる。
* 仮想マシンの内部127.0.0.1、仮想マシンの専用ネットワーク 172.20.1.0/24、仮想マシンのホストから、DNSを利用可能

~~~
maho:vagrant-dns maho$ nslookup 
> server 172.20.1.250
Default server: 172.20.1.250
Address: 172.20.1.250#53

> www.mahot.org
Server:		172.20.1.250
Address:	172.20.1.250#53

Name:	www.mahot.org
Address: 172.20.1.100

> www.google.co.jp
Server:		172.20.1.250
Address:	172.20.1.250#53

Non-authoritative answer:
Name:	www.google.co.jp
Address: 172.217.31.131
~~~

## ドメインの逆引き

IPアドレスからFQDNを求めることもできる。

~~~
maho:vagrant-dns maho$ nslookup 
> server 172.20.1.250
Default server: 172.20.1.250
Address: 172.20.1.250#53
> www.mahot.org
Server:		172.20.1.250
Address:	172.20.1.250#53

Name:	www.mahot.org
Address: 172.20.1.100
> 172.20.1.100
Server:		172.20.1.250
Address:	172.20.1.250#53

100.1.20.172.in-addr.arpa	name = www.mahot.org.
> 
~~~

# DNSサーバー内でansible適用方法

サーバーにログインして、直接 ansible-playbookコマンドを実行して設定を更新することもできる

~~~
maho:vagrant-dns maho$ vagrant ssh
[vagrant@server1 ~]$ sudo -s
[root@server1 vagrant]# ansible-playbook -i /vagrant/playbook/bind/hosts /vagrant/playbook/install_bind.yml 

PLAY [Set up Bind DNS] ********************************************************************

TASK [Gathering Facts] ********************************************************************
ok: [server1]

TASK [bind : Set DNS Server to host1] *****************************************************
ok: [server1]

TASK [bind : Install bind] ****************************************************************
ok: [server1]

TASK [bind : Set hostname] ****************************************************************
ok: [server1]

TASK [bind : Set hostname fact] ***********************************************************
ok: [server1]

TASK [bind : Copy named conf file] ********************************************************
ok: [server1]

TASK [bind : Make named directory] ********************************************************
ok: [server1]

TASK [bind : Copy named conf local file] **************************************************
ok: [server1]

TASK [bind : Make zones Directory] ********************************************************
ok: [server1]

TASK [bind : Copy forward file] ***********************************************************
ok: [server1]

TASK [bind : Copy reverse file] ***********************************************************
ok: [server1]

TASK [bind : check if firewalld is running] ***********************************************
ok: [server1]

TASK [bind : Open firewall port] **********************************************************
ok: [server1]

TASK [bind : resolv.conf replace] *********************************************************
ok: [server1]

PLAY RECAP ********************************************************************************
server1        : ok=14   changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   

~~~


## ダイナミックDNS

SELinuxのモードはpermissiveに変更してありますから、nsupdateを使ってAレコードの変更ができます。

ホスト test1.mahot.org の登録

~~~
[root@server1 vagrant]# nsupdate -k /etc/rndc.key 
> server server1.mahot.org
> zone mahot.org
> update add test1.mahot.org. 600 A 172.30.1.136
> send
~~~

登録の確認

~~~
[root@server1 vagrant]# nslookup
> test1.mahot.org
Server:		127.0.0.1
Address:	127.0.0.1#53

Name:	test1.mahot.org
Address: 172.30.1.136
~~~

Aレコードの削除

~~~
[root@server1 vagrant]# nsupdate -k /etc/rndc.key 
> update del test1.mahot.org
> send
~~~

削除の結果確認

~~~
[root@server1 vagrant]# nslookup
> test1.mahot.org
Server:		127.0.0.1
Address:	127.0.0.1#53

** server can't find test1.mahot.org: NXDOMAIN
~~~



# 参考URL
* Ansible で BINDサーバを構築をしてみた,https://note.com/ystk_note/n/n986d7bdd53c4
* Set up Bind server with Ansible, https://mangolassi.it/topic/12877/set-up-bind-server-with-ansible
* bind error creating <zone name>.jnl, https://forums.centos.org/viewtopic.php?t=66096
* How to Setup DNS Server (Bind) on CentOS8/RHEL8, https://www.linuxtechi.com/setup-bind-server-centos-8-rhel-8/
