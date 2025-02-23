[![CircleCI](https://circleci.com/gh/jpmobile/jpmobile/tree/main.svg?style=svg)](https://circleci.com/gh/jpmobile/jpmobile/tree/main)
[![Code Climate](https://codeclimate.com/github/jpmobile/jpmobile/badges/gpa.svg)](https://codeclimate.com/github/jpmobile/jpmobile)

# jpmobile: A Rails plugin for Japanese mobile-phones

## jpmobileとは
携帯電話特有の機能を Rails や Rack middleware で利用するためのプラグイン。 以下の機能を備える。

* 携帯電話のキャリア判別
* 端末位置情報の取得
    * [GeoKit](https://github.com/geokit/geokit) との連携

* 端末製造番号、契約者番号等の取得
* IPアドレスの検証(キャリアが公開しているIPアドレス帯域からのアクセスか判定)
    * IPアドレスの検証には
        [jpmobile-ipaddresses](http://github.com/jpmobile/jpmobile-ipaddresses
        ) が必要です。

* ディスプレイ情報(画面サイズ、ブラウザ画面サイズ、カラー・白黒、色数)の取得
    * ディスプレイ情報の取得には
        [jpmobile-terminfo](http://github.com/jpmobile/jpmobile-terminfo)
        が必要です。

* 文字コード変換機能／絵文字のキャリア間相互変換
* メールの送信
    * 絵文字と漢字コードの変換

* メールの受信(experimental)
    * 絵文字と漢字コードの変換

また Rails 5.0 に以下の機能を追加する
* ビューへの自動振分け
* 位置情報取得などのリンクヘルパーの追加
* セッションIDをフォーム／リンクに付与(Trans SID)

他のバージョンの Rails については [Versions : Jpmobile vs
Rails](https://github.com/jpmobile/jpmobile/wiki/Version-:-Jpmobile-vs-Rails)
を参照。

## インストール
### gemでインストールする場合
```shell
% gem install jpmobile
```

#### IPアドレス検証が必要な場合
```shell
% gem install jpmobile-ipaddresses
```

#### ディスプレイ情報を取得する必要がある場合
```shell
% gem install jpmobile-terminfo
```

## 使用例

### 携帯電話の識別
環境変数 `ENV['rack.jpmobile']` にキャリアクラスのインスタンスが格納されています。また Rack::Request#mobile
としても取得可能です。

#### キャリアの識別

```ruby
case request.mobile
when Jpmobile::Mobile::Docomo
  # for DoCoMo
when Jpmobile::Mobile::Au
  # for au
when Jpmobile::Mobile::Softbank
  # for SoftBank
when Jpmobile::Mobile::Willcom
  # for Willcom
when Jpmobile::Mobile::Emobile
  # for EMOBILE
else
  # for PC
end
```

あるいは
```ruby
if request.mobile.is_a?(Jpmobile::Mobile::Docomo)
  # for DoCoMo
end
```

#### ビューの中で一部を切替える例
```ruby
<% if request.mobile? %>
  携帯電話からのアクセスです。
<% else %>
  携帯電話からのアクセスではありません。
<% end %>

<% if request.smart_phone? %>
  スマートフォンからのアクセスです。
<% else %>
  スマートフォンからのアクセスではありません。
<% end %>

<% if request.tablet? %>
  タブレットからのアクセスです。
<% else %>
  タブレットからのアクセスではありません。
<% end %>
```

#### 別に用意した携帯電話用コントローラへリダイレクトする例
```ruby
class PcController < ApplicationController
  before_action :redirect_if_mobile

  def index
  end

  private
  def redirect_if_mobile
    if request.mobile?
      pa = params.dup
      pa[:controller] = "/mobile"
      redirect_to pa
    elsif request.smart_phone?
      pa = params.dup
      pa[:controller] = "/smart_phone"
      redirect_to pa
    end
  end
end

class MobileController < ApplicationController
end
```

### 位置情報の取得
Rack::Request#mobile.position に位置情報が格納されます。

```ruby
@latitude   = request.mobile.position.lat
@longuitude = request.mobile.position.lon
```

#### [GeoKit](http://geokit.rubyforge.org) との連携

vendor/plugins/geokit以下にGeoKitがインストールされていると、Jpmobile::PositionにGeoKit::Mappableがincludeされる。したがって、

```ruby
request.mobile.position.distance_to('札幌駅')
```

とすることで、端末と「札幌駅」との距離を求めることができる。詳細は https://www.rubydoc.info/github/geokit/geokit/master/frames
参照。

### 端末情報の取得

端末側から通知されている場合、request.mobile.ident で 契約に固有の識別子もしくは端末の製造番号を取得できる。
両方存在する場合は契約に固有のIDが優先される。

* 契約に固有のID (request.mobile.ident_subscriber)
    * au: EZ番号(サブスクライバ番号)
    * DoCoMo: FOMAカード製造番号
    * EMOBILE: EMnet対応端末から通知されるユニークなユーザID

* 端末製造番号 (request.mobile.ident_device)
    * DoCoMo: 端末製造番号(FOMA, MOVA)
    * SoftBank: 製造番号

### IPの検証
キャリアが公開しているIPアドレス帯域からのアクセスか判定する。
```ruby
request.mobile.valid_ip?
```

ただし [jpmobile-ipaddresses](http://github.com/jpmobile/jpmobile-ipaddresses)
がインストールされていないか、スマートフォンの場合は必ずfalseとなる。

### 端末の画面サイズ
request.mobile.display で Jpmobile::Display クラスのインスタンスが返る。

```ruby
画面幅 <%= request.mobile.display.width %>
画面高さ <%= request.mobile.display.height %>
```

ただし [jpmobile-terminfo](http://github.com/jpmobile/jpmobile-terminfo)
がインストールされていない場合はエラーとなるので注意が必要。

### 文字コード変換機能／絵文字のキャリア間相互変換

jpmobileを読み込むとDoCoMo、Au、SoftBankの絵文字を透過的に扱うことができる。

* Rails の場合は vendor/plugins に配置し、下記の設定を追加することで有効になる。
    ```ruby
    # Rack middleware を追加するメソッド
    Rails.application.config.jpmobile.mobile_filter
      or
    Jpmobile.config.mobile_filter
    ```

    * 下記の設定を追加することで、<form> タグの accept-charset が変更される。

        ```ruby
        # <form accept-charset="Shift_JIS" ...> などに変更する
        Rails.application.config.jpmobile.form_accept_charset_conversion = true
          or
        Jpmobile.config.form_accept_charset_conversion = true
        ```

    * Andriod/iPhone では Google 絵文字や Unicode 6.0
        絵文字が使われています。下記の設定を追加すると、互換性をもたせるために3キャリアの絵文字に変換することができます。また表示の変換も可能です。

        ```ruby
        Rails.application.config.jpmobile.smart_phone_emoticon_compatibility = true
          or
        Jpmobile.config.smart_phone_emoticon_compatibility = true
        ```

携帯電話上では特に問題とならない。PCブラウザでテストする際に問題となるためのオプション。

* Sinatra の場合は下記のように指定する。
    ```ruby
    $LOAD_PATH << './lib/jpmobile/lib'
    require 'jpmobile'
    require 'jpmobile/rack'

    use Jpmobile::Rack::MobileCarrier
    use Jpmobile::Rack::ParamsFilter
    use Jpmobile::Rack::Filter

    get '/' do
      erb :index
    end
    ```

Rails のみ半角・全角の自動変換フィルターが用意されている。用いるには
    ```ruby
    class MyController
      hankaku_filter
    end
    ```

と指定する。またtextareaやhidden/text/passwordのinputタグで半角に変換したくない場合は :input => true
を指定する。このときNokogiriが必要となるため、Gemfileに追記してインストールする必要がある。
    ```ruby
    class MyController
      hankaku_filter :input => true
    end
    ```

Jpmobile内では、各キャリアの絵文字はUnicode私的領域上にマッピングされ、管理される。
このとき、DoCoMo、Auは公式サイト記載のマッピングが使用される。
ただしSoftBankはAuとの重複を避けるため、公式のマッピングに0x1000加算しU+F001以降に割り当てる。
テンプレート内ではUTF-8でエンコードするか、数値文字参照の&#xHHHH;形式で指定する。

絵文字は送出時に内蔵の変換表に基づいて変換され、携帯電話のエンコーディングにあわせて送出される。
携帯電話から受信した絵文字は上記マッピングに基づいてUTF-8でparamsに渡される。

* DoCoMo、Auとの通信時にはShift_JIS、SoftBankとの通信時にはUTF-8が使用される。
* :hankaku=>true指定時は、カタカナは半角カナに変換されて送出される。携帯電話から送られた半角カナは全角カナに変換される。
* 絵文字はキャリアにあわせて変換されて送出される。
* 携帯電話からの絵文字はUnicode私的領域にマップされ、UTF-8でparamsに格納される。

### ビューの自動振り分け
ビューの自動振り分けを行うには、以下の設定が必要です。

```ruby
class ApplicationController < ActionController::Base
  include Jpmobile::ViewSelector
end
```

DoCoMo携帯電話からアクセスすると、
* index_mobile_docomo.html.erb
* index_mobile.html.erb
* index.html.erb

の順でテンプレートを検索し、最初に見付かったテンプレートが利用される。
Auの場合は、index_mobile_au.html.erb、Softbankの場合はindex_mobile_softbank.html.erbが最初に
検索される。

またiPhoneからアクセスすると、
* index_smart_phone_iphone.html.erb
* index_smart_phone.html.erb
* index.html.erb

の順でテンプレートを検索する。 Androidの場合はindex_smart_phone_android.html.erb、Windows
Phoneの場合はindex_smart_phone_windows_phone.html.erbが最初に検索される。

自動振り分けを無効化するには、アクションにおいて以下のように設定する

```ruby
def index
  disable_mobile_view!
end
```

### 位置情報の取得用リンクの生成

以下のようなコードで、端末に位置情報を要求するリンクを出力する。

```ruby
<%= get_position_link_to("位置情報を取得する", :action=>:gps) %>
```

### セッションIDの付与(Trans SID)
#### Cookie非対応携帯だけに付与する

```ruby
class MyController
  trans_sid
end
```

#### PCにも付与する

```ruby
class MyController
  trans_sid :always
end
```

trans_sid 機能を使う場合には、cookie session store を使用することができず、
active_record_storeのgemパッケージが必要です。 また Rails 5.0 では Cookie
が使える場合にはそちらが優先されてしまうため、:always を指定した場合に問題になる場合があります。 trans_sid を使用する際には、例えば
config/initializers/session_store.rb で

```ruby
Rails.application.config.session_store :active_record_store
Rails.application.config.session_options = {:cookie_only => false}
```

として active_record_store を使用するように設定し、:cookie_only => false として Cookie
を優先しないように設定します。 このとき ApplicationController において protect_from_forgery の :secret
を指定するか、 あるいは protect_from_forgery を解除する必要があるでしょう。

また、

```ruby
link_to "hoge", "/controller/action/id"
```

のようにリンク先を直接指定するとセッションIDは付加されません。

```ruby
link_to "hoge", :controller => "controller", :action => "action", :id => "id"
```

のように指定する必要があります。

### メールの送受信
メールにて絵文字や漢字コードの変換、ビューの自動振り分けを行なうには Jpmobile::Mailer::Base を使う。

```ruby
class MobileMailer < Jpmobile::Mailer::Base
  default :from => "info@jpmobile-rails.org"

  def registration(to)
    mail(:to => to)
  end

  def receive(email)
    email
  end
end
```

* デフォルトで ISO-2022-JP エンコードで送信
* docomo/SoftBank には Shift_JIS で送信
* au は ISO-2022-JP エンコードで送信
* ビューやレイアウトの自動振り分け
    * docomo であれば
        app/views/mobile_mailer/registration_mobile_docomo.html.erb など

* 受信に関しては、docomo/SoftBank でテストできていないため、実験版とします。
    * テストして問題あれば随時報告してください。

* オプションに :decorated => true を追加すると、各キャリアのデコメに適したフォーマットで送信します。
    * ただし docomo/SoftBank でテスト出来ていないため、実験版とします。

## jpmobileの開発方法

jpmobileの開発に関しては
[こちら](https://github.com/jpmobile/jpmobile/blob/main/CONTRIBUTING.md) へ

## リンク

* http://jpmobile-rails.org

## 作者

Copyright 2006-2012 (c) Yoji Shidara, under MIT License.

Shin-ichiro OGAWA <rust.stnard@gmail.com>, Yoji Shidara <dara@shidara.net>
