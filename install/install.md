# ceph-ansibleを使ったRed Hat Ceph Storage 3クラスターのデプロイ (コンテナデプロイ編)

## はじめに

RHCS2からcephクラスターのデプロイはceph-ansibleを利用するようになりました(ceph-deployはdeprecated)。  
そこでceph-ansibleを使ったクラスターのデプロイ手順を記載します。  
今回はceph daemonをコンテナとしてデプロイするスタイルとします。

次のようなcephクラスターをイメージしてデプロイする手順を記載します。


![クラスターイメージ](https://github.com/tutsunom/rhcephtest/blob/master/install/cluster_container.png)

---

## 事前セットアップ

クラスターをデプロイする前に各ノードで事前のセットアップ作業が必要になります。  
これも負荷がバカにならない作業なので、mgmtノードからansibleを使ってこれらの設定も自動で行います。

- 全ノードのSSH構成
- 名前解決
- 時刻同期
- ファイアウォールのポート開放
  - MON : tcp/6789,6800-7300
  - OSD : tcp/6800-7300
  - RGW : tcp/7480
- RHCSサブスクリプションをアタッチ
- yumリポジトリの有効化
  - Mgmt : rhel-7-server-rpms, rhel-7-server-extras-rpms, rhel-7-server-rhceph-3-tools-rpms
  - MON : rhel-7-server-rpms, rhel-7-server-extras-rpms, rhel-7-server-rhceph-3-mon-rpms
  - OSD : rhel-7-server-rpms, rhel-7-server-extras-rpms, rhel-7-server-rhceph-3-osd-rpms
  - RGW : rhel-7-server-rpms, rhel-7-server-extras-rpms, rhel-7-server-rhceph-3-tools-rpms
- 時刻同期(ntp)

### 1. mgmtノードにceph-ansibleをインストール
```
[root@ceph-mgmt ~]# subscription-manager register --username=$CDN_USERNAME --password=$CDN_PASSWORD
[root@ceph-mgmt ~]# subscription-manager reflesh
[root@ceph-mgmt ~]# subscription-manager list --available --all --matches="*Ceph*"
[root@ceph-mgmt ~]# subscription-manager attach --pool=$POOL_ID

[root@ceph-mgmt ~]# subscription-manager repos --disable='*' --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rhceph-3-tools-rpms
[root@ceph-mgmt ~]# yum update

[root@ceph-mgmt ~]# yum -y install ceph-ansible
```
### 2. ansible hostsファイルを編集
```
[root@ceph-mgmt ~]# vi /etc/ansible/hosts
[mons]
ceph-01
ceph-02
ceph-03
[mgrs]
ceph-01
ceph-02
ceph-03
[osds]
ceph-01
ceph-02
ceph-03
ceph-04
[rgws]
ceph-04
```

### 3. Passwordless sshのためmgmtノードから各ノードにssh keyの配布
```
[root@ceph-mgmt ~]# ssh-keygen -f /root/.ssh/id_rsa -N ‘’
[root@ceph-mgmt ~]# ansible all -m authorized_key -a "user=root key='{{ lookup('file', '/root/.ssh/id_rsa.pub') }}'" --ask-pass
SSH password: $HOST_PASSWORD
ceph-01 | SUCCESS => {
	"changed": true,
	"exclusive": false,
	"key": "ssh-rsa 
...
[root@ceph-mgmt ~]# ansible all -m ping    ## 確認
```

### 4. RGW/MON/OSDノードのセットアップのためのplaybookを作成して実行
```
[root@ceph-mgmt ~]# vi setup.yml
---
## 全ノード共通の設定
-  hosts: all
   user: root
   tasks:
     - name: register system to RHSM and attach subscription
       redhat_subscription:
         username: $CDN_USERNAME
         password: $CDN_PASSWORD
         pool_id: $POOL_ID
         state=present

## MONの設定
-  hosts: mons
   user: root
   tasks:
     - name: disable all repository and enable rhel-7-server-rpms, rhel-7-server-extras-rpms and rhel-7-server-rhceph-3-mon-rpms
       shell: subscription-manager repos --disable '*' --enable rhel-7-server-rpms --enable rhel-7-server-extras-rpms --enable rhel-7-server-rhceph-3-mon-rpms
     - name: accept 6789/tcp port on firewall
       firewalld:
         port: 6789/tcp
         zone: public
         permanent: true
         state: enabled
     - name: accept 6800-7300/tcp port on firewall
       firewalld:
         port: 6700-7300/tcp
         zone: public
         permanent: true
         state: enabled

## OSDの設定
-  hosts: osds
   user: root
   tasks:
     - name: disable all repository and enable rhel-7-server-rpms, rhel-7-server-extras-rpms and rhel-7-server-rhceph-3-osd-rpms
       shell: subscription-manager repos --disable '*' --enable rhel-7-server-rpms --enable rhel-7-server-extras-rpms --enable rhel-7-server-rhceph-3-osd-rpms
     - name: accept 6800-7300/tcp port on firewall
       firewalld:
         port: 6700-7300/tcp
         zone: public
         permanent: true
         state: enabled

## RGWの設定
-  hosts: rgws
   user: root
   tasks:
     - name: disable all repository and enable rhel-7-server-rpms, rhel-7-server-extras-rpms and rhel-7-server-rhceph-3-tools-rpms
       shell: subscription-manager repos --disable '*' --enable rhel-7-server-rpms --enable rhel-7-server-extras-rpms --enable rhel-7-server-rhceph-3-tools-rpms
     - name: accept 7480/tcp port on firewall
       firewalld:
         port: 7480/tcp
         zone: public
         permanent: true
         state: enabled

[root@ceph-mgmt ~]# ansible-playbook setup.yml
```

failed無しでplaybookが実行されたら準備完了です。

---

## ceph-ansibleの変数用YAMLファイルの作成

ceph-ansibleで必要となる変数はたくさんあります。これらはYAMLファイルに記述され、これがcephクラスターの基本構成を決めます。  
ここではクラスターを構成する必要最小限の項目だけ取り上げて作成しますが、他の項目も興味がある方は後述される各YAMLのサンプルファイルに説明がありますのでご覧下さい。

### 1. keyring用のディレクトリを作成
```
[root@ceph-mgmt ~]# mkdir ~/ceph-ansible-keys
[root@ceph-mgmt ~]# ln -s /usr/share/ceph-ansible/group_vars /etc/ansible/group_vars
```

### 2. all.ymlの作成
all.ymlはクラスター全体に共通する項目が格納されるYAMLファイルです。  
サンプルの`/usr/share/ceph-ansible/group_vars/all.yml.sample`をコピーしてall.ymlを作ります。適宜コメントマーカーを外して編集します。
```
[root@ceph-mgmt ~]# cd /usr/share/ceph-ansible/
[root@ceph-mgmt ceph-ansible]# cp group_vars/all.yml.sample group_vars/all.yml
[root@ceph-mgmt ceph-ansible]# vi group_vars/all.yml		#下記のような項目を設定する
[root@ceph-mgmt ceph-ansible]# grep -v -e '^\s*#' -e '^\s*$' all.yml
---
dummy:
fetch_directory: ~/ceph-ansible-keys
ceph_origin: repository
ceph_repository: rhcs
ceph_repository_type: cdn
ceph_rhcs_version: 3
monitor_interface: bond-pub
public_network: "192.168.30.0/24"
cluster_network: "192.168.40.0/24"
journal_size: 102400
radosgw_interface: bond-pub
ceph_docker_image: rhceph/rhceph-3-rhel7
containerized_deployment: true
ceph_docker_registry: registry.access.redhat.com
```

### 3. osds.ymlの作成
`osds.yml`はOSDの項目が格納されるYAMLファイルです。OSDはdataとjournalの配置など設計によっていろいろなパターンの構成を取ることができますが、今回はシンプルにuserdataとjournalを同居させて全OSDに分散するパターンで作ります。  
`/usr/share/ceph-ansible/group_vars/osds.yml.sample`をコピーして`osds.yml`を作って編集します。
```
[root@ceph-mgmt ~]# cd /usr/share/ceph-ansible/
[root@ceph-mgmt ceph-ansible]# cp group_vars/osds.yml.sample group_vars/osds.yml
[root@ceph-mgmt ceph-ansible]# vi group_vars/osds.yml	#下記のような項目を設定する
[root@ceph-mgmt ceph-ansible]# grep -v -e '^\s*#' -e '^\s*$' osds.yml
---
dummy:
osd_scenario: collocated
devices:
  - /dev/nvme0n1
  - /dev/nvme0n2
  - /dev/nvme0n3
  - /dev/nvme0n4  
```

### 4. rgws.ymlの作成
`rgws.yml`はRGWの項目が格納されるYAMLファイルです。`rgws.yml`ではいろいろと設定することができますが、ここではデフォルトのままで設定します。  
`/usr/share/ceph-ansible/group_vars/rgws.yml.sample`をコピーして`rgws.yml`を作り、編集せずに進みます。
```
[root@ceph-mgmt ~]# cd /usr/share/ceph-ansible/
[root@ceph-mgmt ceph-ansible]# cp group_vars/rgws.yml.sample group_vars/rgws.yml
```

### 5. playbookの実行
ceph-ansibleのplaybookは`/usr/share/ceph-ansible/site.yml.sample`をコピーして`site.yml`として作ります。  
`site.yml`自身は編集する必要がなく、そのまま`ansible-playbook`コマンドで指定してplaybookを実行します。  
```
[root@ceph-mgmt ~]# cd /usr/share/ceph-ansible/
[root@ceph-mgmt ceph-ansible]# cp site.yml.sample site.yml
[root@ceph-mgmt ceph-ansible]# echo "retry_files_save_path = ~/" >> /etc/ansible/ansible.cfg	#retryファイルのパスをホームディレクトリに指定
[root@ceph-mgmt ceph-ansible]# ansible-playbook site.yml
...
...
PLAY RECAP ***********************************************************************************************************
ceph-01		: ok=97   changed=22   unreachable=0    failed=0   
ceph-02		: ok=84   changed=18   unreachable=0    failed=0   
ceph-03		: ok=89   changed=19   unreachable=0    failed=0   
ceph-04		: ok=45   changed=14   unreachable=0    failed=0   
```

---

## クラスターの動作確認

### ステータスをチェックする
無事にplaybookがエラー無く実行完了したら動作確認をしてみます。  
いずれかのmonのコンテナが稼働するノードにログインしてクラスターのステータスを確認します。


```
[root@ceph-01 ~]# docker ps
CONTAINER ID        IMAGE                                                     COMMAND             CREATED             STATUS              PORTS               NAMES
3f6803f03891        registry.access.redhat.com/rhceph/rhceph-3-rhel7:latest   "/entrypoint.sh"    40 hours ago        Up 40 hours                             ceph-osd-ceph-01-nvme0n1
1189a76f6440        registry.access.redhat.com/rhceph/rhceph-3-rhel7:latest   "/entrypoint.sh"    40 hours ago        Up 40 hours                             ceph-osd-ceph-01-nvme0n2
c4b008869a06        registry.access.redhat.com/rhceph/rhceph-3-rhel7:latest   "/entrypoint.sh"    40 hours ago        Up 40 hours                             ceph-osd-ceph-01-nvme0n3
c9ed19d23c5c        registry.access.redhat.com/rhceph/rhceph-3-rhel7:latest   "/entrypoint.sh"    40 hours ago        Up 40 hours                             ceph-osd-ceph-01-nvme0n4
b78be631b351        registry.access.redhat.com/rhceph/rhceph-3-rhel7:latest   "/entrypoint.sh"    40 hours ago        Up 40 hours                             ceph-mgr-ceph-01
f4d59836d3f9        registry.access.redhat.com/rhceph/rhceph-3-rhel7:latest   "/entrypoint.sh"    40 hours ago        Up 40 hours                             ceph-mon-ceph-01
[root@ceph-01 ~]# docker exec -it ceph-mon-ceph-01 bash
[root@ceph-01 /]# ceph status
  cluster:
    id:     5765942e-be0e-4a70-8542-871bf4ced6c4
    health: HEALTH_WARN
            too few PGs per OSD (16 < min 30)
 
  services:
    mon: 3 daemons, quorum ceph-01,ceph-02,ceph-03
    mgr: ceph-02(active), standbys: ceph-03, ceph-01
    osd: 12 osds: 12 up, 12 in
    rgw: 1 daemon active
 
  data:
    pools:   6 pools, 48 pgs
    objects: 208 objects, 3359 bytes
    usage:   971 MB used, xxxxx GB / xxxxx GB avail
    pgs:     48 active+clean

```
`too few PGs per OSD`という内容のWARNINGが出ています。デフォルトで作られているプールに割り振られているPGが少ないようです。  
デフォルトのプールのPGを増やしてもいいのですが、せっかくなので新しいプールを作ってもう一度確認してみます。
```
[root@ceph-01 /]# ceph osd pool create mypool 256 256
pool ‘mypool’ created
[root@ceph-01 /]# ceph status
  cluster:
    id:     5765942e-be0e-4a70-8542-871bf4ced6c4
    health: HEALTH_OK
...
  data:
    pools:   7 pools, 304 pgs
    objects: 208 objects, 3359 bytes
    usage:   977 MB used, xxxxx GB / xxxxx GB avail
    pgs:     304 active+clean
```
クラスター全体でPGの数が増えたため、`HEALTH_OK`になりました。

### プールにオブジェクトをPUTする
次にMONノードからプールにオブジェクトをPUTしてみます。
```
[root@ceph-01 /]# echo "Hello world of ceph." > ~/test.txt

[root@ceph-01 /]# ls -l ~/test.txt
-rw-r--r--. 1 root root 21  8月 09 00:16 ~/test.txt
[root@ceph-01 /]# rados --pool mypool put helloworld ~/test.txt
[root@ceph-01 /]# rados --pool mypool ls
helloworld
[root@ceph-01 /]# rados --pool mypool df
POOL_NAME USED OBJECTS CLONES COPIES MISSING_ON_PRIMARY UNFOUND DEGRADED RD_OPS RD WR_OPS WR     
mypool      21       1      0      3                  0       0        0      0  0      1 1024 
...
```
`test.txt`というファイルを`hellowworld`というオブジェクト名で`mypool`にPUTした結果です。  
`USED`と`OBJECTS`にはそれぞれプールに置かれているオブジェクトのサイズと個数が表示されます。`WR_OPS`と`WR`にはそれぞれ書き込み回数と書き込まれた容量が表示されます。
`COPIES`が3であるのは`mypool`が3x replicationで作られているためです。念の為確認してみましょう。
```
[root@ceph-01 /]# ceph osd pool get mypool size
size: 3
```

### プールからオブジェクトをGETする
別のノードにログインしてプールからオブジェクトをGETしてみます。ここでは ceph-02 にログインします。
```
[root@ceph-01 /]# exit
exit
[root@ceph-01 ~]# ssh ceph-02
[root@ceph-01 ~]# ssh ceph-02
[root@ceph-02 ~]# docker exec -it ceph-mon-ceph-02 bash
[root@ceph-02 /]# rados --pool mypool ls
helloworld
[root@ceph-02 /]# rados --pool mypool get helloworld get.txt
[root@ceph-02 /]# ls -l get.txt
-rw-r--r--. 1 root root   21  8月 09 01:00 get.txt
[root@ceph-02 /]# cat get.txt 
Hello world of ceph.
[root@ceph-02 /]# rados --pool mypool df
POOL_NAME USED OBJECTS CLONES COPIES MISSING_ON_PRIMARY UNFOUND DEGRADED RD_OPS RD   WR_OPS WR   
mypool2     21       1      0      3                  0       0        0      1 1024      1 1024 
...
```
先程とは違うノードから`helloworld`オブジェクトをGETできました。`RD_OPS`と`RD`はそれぞれ読み込み回数と読み込み容量が表示されるところであり、GETしたことによりそれぞれ数値が変わったことが確認できます。

---

## Revision History
2018-08-09: 初版リリース
