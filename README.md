# Ansible Role: Apache 2.x (オフラインインストール用)

Forked from [geerlingguy/ansible-role-apache](https://github.com/geerlingguy/ansible-role-apache)
Edited by Sato Kenta

RHEL/CentOS, Debian/Ubuntu, SLES and Solarisで使用できるApache 2.xのAnsible Roleです。
このロールは、インターネットに接続できない閉域ネットワークでの実行用にカスタマイズされています。

## 前提事項など

SSL/TLSを使用する場合, 証明書ファイルを別途準備する必要があります。`openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout example.key -out example.crt`のようなコマンドを使用して自己証明書を作成するこも可能です。

PHPを使用する場合、別途``等のロールを使用してPHPをインストールするか、PHPがバンドルされたApacheのロールを使用してください。

## 設定可能な変数について

使用可能な変数とそのデフォルト値を以下のように記載します。(デフォルト値は`defaults/main.yml`を参照)

    apache_enablerepo: ""

上記はApacheをインストールする際に使用するレポジトリを指定します (RHEL/CentOS系OSの場合のみ). OSで提供されているレポジトリより新しいバージョンを使用する場合はEPELなどを使用してください。 (`geerlingguy.repo-epel`ロールでインストールできます).

    apache_listen_ip: "*"
    apache_listen_port: 80
    apache_listen_port_ssl: 443

上記はApacheがリッスンするIPアドレスとポートを指定します。リバースプロキシなどを使用して別のサービスから通信を受ける場合はこれを変更してください。

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

Add a set of properties per virtualhost, including `servername` (required), `documentroot` (required), `allow_override` (optional: defaults to the value of `apache_allow_override`), `options` (optional: defaults to the value of `apache_options`), `serveradmin` (optional), `serveralias` (optional) and `extra_parameters` (optional: you can add whatever additional configuration lines you'd like in here).

Here's an example using `extra_parameters` to add a RewriteRule to redirect all requests to the `www.` site:

      - servername: "www.local.dev"
        serveralias: "local.dev"
        documentroot: "/var/www/html"
        extra_parameters: |
          RewriteCond %{HTTP_HOST} !^www\. [NC]
          RewriteRule ^(.*)$ http://www.%{HTTP_HOST}%{REQUEST_URI} [R=301,L]

The `|` denotes a multiline scalar block in YAML, so newlines are preserved in the resulting configuration file output.

    apache_vhosts_ssl: []

No SSL vhosts are configured by default, but you can add them using the same pattern as `apache_vhosts`, with a few additional directives, like the following example:

    apache_vhosts_ssl:
      - servername: "local.dev"
        documentroot: "/var/www/html"
        certificate_file: "/home/vagrant/example.crt"
        certificate_key_file: "/home/vagrant/example.key"
        certificate_chain_file: "/path/to/certificate_chain.crt"
        extra_parameters: |
          RewriteCond %{HTTP_HOST} !^www\. [NC]
          RewriteRule ^(.*)$ http://www.%{HTTP_HOST}%{REQUEST_URI} [R=301,L]

Other SSL directives can be managed with other SSL-related role variables.

    apache_ssl_protocol: "All -SSLv2 -SSLv3"
    apache_ssl_cipher_suite: "AES256+EECDH:AES256+EDH"

The SSL protocols and cipher suites that are used/allowed when clients make secure connections to your server. These are secure/sane defaults, but for maximum security, performand, and/or compatibility, you may need to adjust these settings.

    apache_allow_override: "All"
    apache_options: "-Indexes +FollowSymLinks"

The default values for the `AllowOverride` and `Options` directives for the `documentroot` directory of each vhost.  A vhost can overwrite these values by specifying `allow_override` or `options`.

    apache_mods_enabled:
      - rewrite.load
      - ssl.load
    apache_mods_disabled: []

(Debian/Ubuntu ONLY) Which Apache mods to enable or disable (these will be symlinked into the appropriate location). See the `mods-available` directory inside the apache configuration directory (`/etc/apache2/mods-available` by default) for all the available mods.

    apache_packages:
      - [platform-specific]

The list of packages to be installed. This defaults to a set of platform-specific packages for RedHat or Debian-based systems (see `vars/RedHat.yml` and `vars/Debian.yml` for the default values).

    apache_state: started

Set initial Apache daemon state to be enforced when this role is run. This should generally remain `started`, but you can set it to `stopped` if you need to fix the Apache config during a playbook run or otherwise would not like Apache started at the time this role is run.

    apache_packages_state: present

If you have enabled any additional repositories such as _ondrej/apache2_, [geerlingguy.repo-epel](https://github.com/geerlingguy/ansible-role-repo-epel), or [geerlingguy.repo-remi](https://github.com/geerlingguy/ansible-role-repo-remi), you may want an easy way to upgrade versions. You can set this to `latest` (combined with `apache_enablerepo` on RHEL) and can directly upgrade to a different Apache version from a different repo (instead of uninstalling and reinstalling Apache).

    apache_ignore_missing_ssl_certificate: true

If you would like to only create SSL vhosts when the vhost certificate is present (e.g. when using Let’s Encrypt), set `apache_ignore_missing_ssl_certificate` to `false`. When doing this, you might need to run your playbook more than once so all the vhosts are configured (if another part of the playbook generates the SSL certificates).

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
