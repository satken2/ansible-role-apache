# Ansible Role: Apache 2.x (オフライン用)

Forked from [geerlingguy/ansible-role-apache](https://github.com/geerlingguy/ansible-role-apache)<br>
Edited by Sato Kenta

RHEL/CentOS, Debian/Ubuntu, SLES and Solarisで使用できるApache 2.xのAnsible Roleです。
このロールは、インターネットに接続できない閉域ネットワークでの実行用にカスタマイズされています。

## 前提事項など

SSL/TLSを使用する場合, 証明書ファイルを別途準備する必要があります。`openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout example.key -out example.crt`のようなコマンドを使用して自己証明書を作成するこも可能です。

PHPを使用する場合、別途``等のロールを使用してPHPをインストールするか、PHPがバンドルされたApacheのロールを使用してください。

## 設定可能な変数について

使用可能な変数とそのデフォルト値を以下に記載します。(`defaults/main.yml`を参照)

### 基本設定

    apache_enablerepo: ""

上記はApacheをインストールする際に使用するレポジトリを指定します (RHEL/CentOS系OSの場合のみ). OSで提供されているレポジトリより新しいバージョンを使用する場合はEPELなどを使用してください。 (`geerlingguy.repo-epel`ロールでインストールできます).

    apache_listen_ip: "*"
    apache_listen_port: 80
    apache_listen_port_ssl: 443

上記はApacheがリッスンするIPアドレスとポートを指定します。リバースプロキシなどを使用して別のサービスから通信を受ける場合はこれを変更してください。

    apache_state: started

このロールの実行時に適用されるApacheデーモンの初期状態を設定します。特に要件がない場合`started`に設定してください。ただし、Playbookの実行中にApache設定を修正する必要がある場合や、このロールの実行後に何らかの理由でApacheを開始したくない場合は、`stopped`に設定することもできます。

### 仮想ホストの設定

    apache_create_vhosts: true
    apache_vhosts_filename: "vhosts.conf"
    apache_vhosts_template: "vhosts.conf.j2"

Trueに設定した場合、ロールでvhostsファイルを生成(後述)し、Apacheのコンフィグフォルダに配置することができます。Falseに設定し、自分でvhosts定義ファイルを作成してコンフィグフォルダに配置することもできます。
You can also override the template used and set a path to your own template, if you need to further customize the layout of your VirtualHosts.

    apache_remove_default_vhost: false

Debian/Ubuntuではデフォルトのvirtualhostが最初からApacheのコンフィグに含まれていますが、このパラメータを`true`に設定すればその設定を削除することができます。


    apache_global_vhost_settings: |
      DirectoryIndex index.php index.html
      # Add other global settings on subsequent lines.

`apache_create_vhosts`がTrueの場合でも、この変数を使用することでRoleが生成するvhostsファイルを上書きすることができます。デフォルトでは、単にDirectoryIndexの設定だけを行います。

    apache_vhosts:
      # Additional optional properties: 'serveradmin, serveralias, extra_parameters'.
      - servername: "local.dev"
        documentroot: "/var/www/html"

このように、vhostsに設定したいプロパティを記載します。設定できる値は`servername`(必須)、` documentroot`(必須)、 `allow_override`(オプション：デフォルトは` apache_allow_override`)、 `options`(オプション：デフォルトは`apache_options`)、` serveradmin`(オプション)、 `serveralias`(オプション)、`extra_parameters`(オプション：ここに必要な構成行を追加できます)です。<br>
複数のvhostを設定する場合はYAMLのリスト書式で追記してください。

別の例として、`extra_parameters`を使用してRewriteRuleを追加し、すべてのリクエストを` www.`サイトにリダイレクトする例を示します。

      - servername: "www.local.dev"
        serveralias: "local.dev"
        documentroot: "/var/www/html"
        extra_parameters: |
          RewriteCond %{HTTP_HOST} !^www\. [NC]
          RewriteRule ^(.*)$ http://www.%{HTTP_HOST}%{REQUEST_URI} [R=301,L]


`|`はYAMLにおいて複数行のスカラーブロックを示す書式であり、ここに記載した改行文字は生成される構成ファイルに反映されます。

    apache_vhosts_ssl: []

デフォルトではSSLvhostは設定されていませんが、以下の例のようにいくつかの追加のディレクティブを使用して、 `apache_vhosts`と同様にSSLvhostをYAMLのリスト形式で追加できます。

    apache_vhosts_ssl:
      - servername: "local.dev"
        documentroot: "/var/www/html"
        certificate_file: "/home/vagrant/example.crt"
        certificate_key_file: "/home/vagrant/example.key"
        certificate_chain_file: "/path/to/certificate_chain.crt"
        extra_parameters: |
          RewriteCond %{HTTP_HOST} !^www\. [NC]
          RewriteRule ^(.*)$ http://www.%{HTTP_HOST}%{REQUEST_URI} [R=301,L]

他のSSLディレクティブは、他のSSL関連の変数で設定できます。

### SSL関連設定

    apache_ssl_protocol: "All -SSLv2 -SSLv3"
    apache_ssl_cipher_suite: "AES256+EECDH:AES256+EDH"

クライアントがサーバーに安全に接続するときに使用/許可されるSSLプロトコルと暗号化方式を設定します。これらの設定はデフォルトの状態で十分なセキュリティが担保されますが、セキュリティ、パフォーマンス、互換性でチューニングが必要な場合はこれらの設定を変更することができます。

    apache_allow_override: "All"
    apache_options: "-Indexes +FollowSymLinks"

各仮想ホストの `documentroot`ディレクトリの` AllowOverride`および `Options`ディレクティブのデフォルト値。仮想ホストは、 `allow_override`または` options`を指定することにより、これらの値を上書きできます。

    apache_ignore_missing_ssl_certificate: true

vhost証明書が存在する場合にのみSSLvhostを作成する場合(Let’s Encryptを使用する場合など)は、 `apache_ignore_missing_ssl_certificate`を` false`に設定します。この設定を行うときは、すべてのvhostが正しく設定されるように、Playbookを複数回実行する必要がある場合があります。

### Mod関連設定

    apache_mods_enabled:
      - rewrite.load
      - ssl.load
    apache_mods_disabled: []

(Debian / Ubuntuのみ)どのApache modを有効または無効にするか(これらの設定値を元に適切な場所にシンボリックリンクが作成されます)。使用可能なすべてのmodについては、apache構成ディレクトリ内の`mods-available`ディレクトリ(デフォルトでは`/etc/apache2/mods-available`)を参照してください。

    apache_packages:
      - [platform-specific]

インストールするApacheパッケージのリストです。これは、デフォルトでRedHatまたはDebianベースのシステム用のプラットフォーム固有のパッケージのセットになります。(デフォルト値については、`vars/RedHat.yml`および`vars/Debian.yml`を参照)。

    apache_packages_state: present

*ondrej/apache2*などの追加レポジトリを有効にしている場合、この設定を`latest`に設定することで、別のレポジトリからでも直接Apacheを最新状態にアップグレードすることができます。

### パフォーマンス設定

    apache_start_servers: 3
    apache_server_limit: 200
    apache_min_spare_servers: 3
    apache_max_spare_servers: 5
    apache_max_request_workers: 175
    apache_max_connection_per_child: 100
    apache_max_request_per_child: 20


## .htaccessを使用したBasic認証の設定について

Basic認証を設定する場合、Basin認証の設定を含むコンフィグのテンプレートを作成するか、あるいは`extra_parameters`応用することで実現できます。以下に例を示します。

    extra_parameters: |
      <Directory "/var/www/password-protected-directory">
        Require valid-user
        AuthType Basic
        AuthName "Please authenticate"
        AuthUserFile /var/www/password-protected-directory/.htpasswd
      </Directory>

VirtualHostディレクティブ配下の全てをパスワードで保護したい場合、`Directory`ブロックの代わりに`Location`ブロックを使用して以下のようにします。

    <Location "/">
      Require valid-user
      ....
    </Location>

この場合、別途`.htpasswd`ファイルを作成してPlaybookに含める必要があります。

## 依存Role

なし

## Playbook例

    - hosts: webservers
      vars_files:
        - vars/main.yml
      roles:
        - { role: geerlingguy.apache }

*`vars/main.yml`の設定例*:

    apache_listen_port: 8080
    apache_vhosts:
      - {servername: "example.com", documentroot: "/var/www/vhosts/example_com"}

## ライセンス

MIT / BSD
