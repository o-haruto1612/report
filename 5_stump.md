# 5. スタンプ機能

変更を加える前のmastodonには、ある投稿に対してリアクションするには以下の写真のように返信、共有、お気に入り、ブックマークの機能がある。  


![5-1.png](/image/5-1.png)
変更する前のmastodonの投稿とリアクションボタン  

初めの目標ではスタンプを自作して使えるようにすることを考えたが、想像以上に難しかったので、既存の関数を参考にしてグッドボタンとバッドボタンを追加した。

## スタンプを押した時の挙動について

スタンプを追加する上で、まず初めにタイムライン上でお気に入り(星型)のスタンプをクリックした時にどのような挙動になるのか探索した。以下ではお気に入り登録をした時の挙動を追う。

### フロントエンド

初めにreactで書かれたフロントエンドの挙動を追う。
探索する時は、手探りで関数を探したり、デバッガを用いて遷移を追ったりした。

まず、お気に入りボタンの表示はstatus_action_bar.jsx内で定義されており、

```
<IconButton className='status__action-bar__button star-icon' animate active={status.get('favourited')} title={intl.formatMessage(messages.favourite)} icon='star' onClick={this.handleFavouriteClick} counter={withCounters ? status.get('favourites_count') : undefined} />
```
上のような定義によって画面に表示される。IconButtonというのはicon_button.tsxで定義された型であり、status.get(favourited)はバックエンドからjson形式で送られたデータでお気に入りかどうかを示すbool値が入り、ボタンがクリックされるとhandleFavouriteClick関数が呼び出される。この関数は同じファイル内に定義されており、以下のようである。
```
handleFavouriteClick = () => {
const { signedIn } = this.context.identity;

if (signedIn) {
    this.props.onFavourite(this.props.status);
} else {
    this.props.onInteractionModal('favourite', this.props.status);
}
};
```
今回の場合はstatus_container.jsxで定義されたonFavourite関数が呼び出され、
```
onFavourite (status) {
if (status.get('favourited')) {
    dispatch(unfavourite(status));
} else {
    dispatch(favourite(status));
}
},
```
この関数では、お気に入りの状態であればunfavourite関数を、そうでなければfavourite関数を呼び出し、クリックするたびにオンオフを切り替えていることがわかる。お気に入り登録する場合はfavourite関数が呼び出される。

favourite関数はinteraction.jsで定義され、以下のような内容である。
```
export function favourite(status) {
  return function (dispatch, getState) {
    dispatch(favouriteRequest(status));
    api(getState).post(`/api/v1/statuses/${status.get('id')}/favourite`).
      then(function (response) {
        dispatch(importFetchedStatus(response.data));
        dispatch(favouriteSuccess(status));
      }).catch(function (error) {
        dispatch(favouriteFail(status, error));
      });
  };
}
```
簡単に説明するとこの関数は、バックエンドにお気に入り登録をするという情報をパスを指定して送ってバックエンドの関数を呼び出して、その返り値を受け取って処理するというものである。ちなみに、unfavourite関数も同じファイルで定義され、これはお気に入りを削除するような処理をするような関数である。

バックエンドの関数を呼び出す際はさらにapi.rbというファイルに書かれているようにパスを変換して読み出している。api.rbの今回関連する部分を抜き出すと以下のようになっている。
```
namespace :api, format: false do
  # OEmbed
  get '/oembed', to: 'oembed#show', as: :oembed

  # JSON / REST API
  namespace :v1 do
    resources :statuses, only: [:create, :show, :update, :destroy] do
      scope module: :statuses do
        resource :favourite, only: :create
```
このルーティングによってReactのfavourite関数は、favourites_controller.rbに書かれたApi::V1::FavouritesControllerクラスのcreate関数を呼び出す。

### バックエンド
ここからはバックエンドの挙動を示す。

フロントエンドからApi::V1::FavouritesControllerクラスのcreate関数を呼び出された。関数の内容は以下のようである。
```
def create
    set_status
    FavouriteService.new.call(current_account, @status)
    render json: @status, serializer: REST::StatusSerializer
end
```

関数内１行目のset_status関数では、@statusにお気に入りをした投稿の情報を代入し、２行目のFaviuriteServiceを呼び出している部分は後ほど詳しく書くが、データベースに情報を追加し、３行目ではフロントエンドに@statusの情報をjson形式で返している。おそらくfavourite関数がこの返り値を受け取っているのだろう。

2行目のクラスや関数の定義はfavourite_service.rbに書かれている。関数の内容は以下のようで、
```
def call(account, status)
    authorize_with account, status, :favourite?
    favourite = Favourite.find_by(account: account, status: status)
    return favourite unless favourite.nil?

    favourite = Favourite.create!(account: account, status: status)
    Trends.statuses.register(status)
    create_notification(favourite)
    bump_potential_friendship(account, status)
    favourite
end
```
長くて様々な操作をしているが、今回着目するのは
```
favourite = Favourite.create!(account: account, status: status)
```
この部分でFavouriteクラスのcreate!関数(この関数は今回のコードで定義されているのではなく、Favouriteの親のApplicationRecord)を呼び出すことで、データベースに新しくお気に入り登録した情報を追加できる。

### お気に入りを削除した時

削除した時の関数も概ね定義のされ方はお気に入り登録した時のファイルに書き込まれていたりしており、挙動が似ている。フロントエンドからApi::V1::FavouritesControllerクラスのdestroy関数が呼び出され、(以下)
```
  def destroy
    fav = current_account.favourites.find_by(status_id: params[:status_id])

    if fav
      @status = fav.status
      count = [@status.favourites_count - 1, 0].max
      UnfavouriteWorker.perform_async(current_account.id, @status.id)
    else
      @status = Status.find(params[:status_id])
      count = @status.favourites_count
      authorize @status, :show?
    end

    relationships = StatusRelationshipsPresenter.new([@status], current_account.id, favourites_map: { @status.id => false }, attributes_map: { @status.id => { favourites_count: count } })
    render json: @status, serializer: REST::StatusSerializer, relationships: relationships
  rescue Mastodon::NotPermittedError
    not_found
  end
```

今回の場合はUnfavouriteWorkerクラスのperform関数(unfavourite_service.rb)が呼び出され、
```
class UnfavouriteWorker
  include Sidekiq::Worker

  def perform(account_id, status_id)
    UnfavouriteService.new.call(Account.find(account_id), Status.find(status_id))
  rescue ActiveRecord::RecordNotFound
    true
  end
end
```
次にUnfavouriteServiceクラスのcall関数(unfavourite_service.rb)が呼び出される。
```
  def call(account, status)
    favourite = Favourite.find_by!(account: account, status: status)
    favourite.destroy!
    create_notification(favourite) if !status.account.local? && status.account.activitypub?
    favourite
  end
```
call関数では消したいお気に入りの情報を取得し、
```
favourite.destroy!
```
の部分でデータベースからお気に入り登録をした履歴を消去する

## スタンプの作成

以上に書いたスタンプの挙動を参考にしてグッドボタン、バッドボタンに用いる関数やクラスを定義してスタンプを押せるようにした。細かいファイルについては多少機能に応じて異なる部分もあるが概ね上述したお気に入りボタンの'favourite'を'thumbsup'や'thumbsdown'に書き換えて作成しただけなのでコードは載せない。

ただ、データベースに情報を登録するところでは新しい操作があったのでその部分については詳しく以下で述べる。

### データベースへの情報登録
お気に入りボタンについてはデータベースはすでに作成してあり、schema.rbに
```
create_table "favourites", force: :cascade do |t|
    t.datetime "created_at", precision: nil, null: false
    t.datetime "updated_at", precision: nil, null: false
    t.bigint "account_id", null: false
    t.bigint "status_id", null: false
    t.index ["account_id", "id"], name: "index_favourites_on_account_id_and_id"
    t.index ["account_id", "status_id"], name: "index_favourites_on_account_id_and_status_id", unique: true
    t.index ["status_id"], name: "index_favourites_on_status_id"
end
```
このようにテーブルを作るように指示されている。このテーブルでは以下のような情報を持つ。

|名前|内容|
|:-------:|:------|
|created_at|作成された日付、定義しなくても自動で作られる|
|updated_at|更新された日付、定義しなくても自動で作られる|
|account_id|お気に入りをしたアカウントのID、一意である|
|status_id|お気に入りをされた投稿のID、一意である|

新しくスタンプを定義したらこのようにデータベースのテーブルを作るように定義しなければいけないが、schema.rbに直接書き込むことはあまりよくないようで、以下に示すような操作をしてマイグレーションファイルを作成し、変更を適用した。

マイグレーションファイルとは、データベースに変更を加える時に作成するファイルでrailsを用いる場合ターミナルで
```
rails generate migration 'クラス名'
```
とするとdb/migrateディレクトリに作成でき、ファイルを編集して
```
rails db:migrate
```
とすると変更が適用され、schema.rbの内容が書き換わる。

今回の実験では、使用しているコンテナの都合上(?)、このコマンドでは動かず、
それぞれ頭に'bundle exec'をつけて
```
bundle exec rails generate migration 'クラス名'
```
```
bundle exec rails db:migrate
```
というコマンドで操作した。

実際にグッドボタンを作成する時を例にすると、
```
bundle exec rails generate migration CreateThumbsup
```
とすると、20231022124735_create_thumbsup.rbというファイルが作成され、このファイルの内容を
```
# frozen_string_literal: true

class CreateThumbsup < ActiveRecord::Migration[7.0]
  def change
    create_table :thumbsups do |t|
      t.bigint :account_id, null: false
      t.bigint :status_id, null: false

      t.timestamps
    end
  end
end
```
と書き込んでaccount_idとstatus_idを持つthumbsupsというテーブルを作成した。このままマイグレーションをaccout_idとstatus_idは一意でなければいけないとエラーが出たので、今度は
```
bundle exec rails generate migration uniqueThumbsup
```
として再び20231023070913_thumbsupunique.rbを作成し、以下のように
```
# frozen_string_literal: true

class Thumbsupunique < ActiveRecord::Migration[7.0]
  disable_ddl_transaction!

  def change
    add_index :thumbsups, [:status_id, :account_id], unique: true, algorithm: :concurrently
  end
end
```
と書き込み、
```
bundle exec rails db:migrate
```
と変更を適用するとエラーが解消され、schema.rb内を確認すると自動でthumbsupsというテーブルができており(以下)、データを登録できるようになった。

```
create_table "thumbsups", force: :cascade do |t|
    t.bigint "account_id", null: false
    t.bigint "status_id", null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["status_id", "account_id"], name: "index_thumbsups_on_status_id_and_account_id", unique: true
end
```

### 表示の変更と機能追加

新しくスタンプの機能を追加できたが、表示的な機能がまだできていないので、定義した。具体的には以下の内容である。
- 色の変化(グッドボタン→水色、バッドボタン→赤色)
- クリックするとスタンプが並んでいるコンテナが開くようにする
- グッドボタンとバッドボタンの排他性(片方がオンになったらもう片方は強制的にオフになる)

#### 色の変化

この時もお気に入りのスタンプを参考に探索して、色や配置の定義はcomponent.scssというファイルで行われていることがわかった。また、chrome上で検証を行い、クリックした時の挙動を調べると、オンになっている時にはアイコンには'active'クラスがついており、またグッドボタン、バッドボタンそれぞれに'thumbsup-icon'、'thumbsdown-icon'というクラスが定義されていることがわかったので、components.scssには以下のように付け足した。
```
.icon-button.thumbsup-icon.active {
  color: #19cdf1;
}

.icon-button.thumbsdown-icon.active {
  color: #ff3a78;
}
```
これでオンになった時に色が変化するようになった。

#### スタンプが入ったコンテナを作る

今は追加で二つしかスタンプを作成していないので煩わしくないが、スタンプが増えた時に一つ一つの投稿に対して全てのスタンプを表示していたら表示を圧迫していないので、クリックしたらスタンプが入っているコンテナを開くようなボタンを作成した。形は下三角形です。

ボタンの定義は、status_action_bar.jsxに書き、
```
<IconButton className='status__action-bar__button caret-down-icon' icon='caret-down' onClick={this.handleCaretdownClick} />
```
となっており、クリックすると
```
  handleCaretdownClick = () => {
    this.setState({ isDivVisible: !this.state.isDivVisible });
  };
```
この関数が呼び出され、isDivVisibleの値が入れ替わり、見えなくなっているdivが見えるようになる。見えるようになっている時にクリックすると、今度は見えなくなる。

このようにしてクリックすると反応して色が変わるボタンが完成した。スタンプを押した時の様子を下の写真に示す。

![5-2.png](./image/5-2.png)
変更後のリアクションボタンの様子

#### グッドボタンとバッドボタンの排他性

グッドとバッドが両方とも押されているという状態は基本的にない方が好ましいので、片方がオンになっている状態でもう片方をオンにすると初めにオンになっている方はオフになるようにした。

バッドボタンが押されてる時にグッドボタンをオンにした時の挙動を考え、変更を加えた。以下にstatus_container.jsxの対応する部分のコードを示す。
```
  onThumbsup (status) {
    if (status.get('thumbsuped')) {
      dispatch(unthumbsup(status));
    } else {
      if(status.get('thumbsdowned')){
        dispatch(unthumbsdown(status));
      }
      dispatch(thumbsup(status));
    }
  },
```

バッドボタンの方も同様に改良したが、何度かボタンを付けたり消したりしてると、表示上では両方ついてしまっていることがあった。原因は突き止め切れていないので、今のコードだとおそらく両方ついているような動作をするという不具合がある。ただ、他の操作をしたり、ページの更新を行うと本来消えるべきスタンプが消えているので、上に示したコードは機能しているが表示だけ変更しない何かがあるのではないかと考える。

### その他

これまでの変更はタイムラインを開いている時だったので、一つの投稿のみを表示した時のスタンプは増えていない状態であった。なので、そのファイルにも少し同じような変更を施し同じような見た目になるようにした。

### パスについて

本文の内容でのコードが入っているファイルのパスを以下に示す。(コード内では異なるディレクトリに同じファイル名で定義されているものもある可能性があるため)

|ファイル名|パス|
|:-------|:------|
|status_action_bar.jsx|app/javascript/mastodon/components/status_action_bar.jsx|
|status_container.jsx|app/javascript/mastodon/containers/status_container.jsx|
|interactions.js|app/javascript/mastodon/actions/interactions.js|
|api.rb|config/routes/api.rb|
|favourites_controller.rb|app/controllers/api/v1/statuses/favourites_controller.rb|
|favourite_service.rb|app/services/favourite_service.rb|
|favourite.rb|app/models/favourite.rb|
|unfavourite_worker.rb|app/workers/unfavourite_worker.rb|
|unfavourite_service.rb|app/services/unfavourite_service.rb|
|schema.rb|db/schema.rb|
|20231022124735_create_thumbsup.rb|db/migrate/20231022124735_create_thumbsup.rb|
|20231023070913_thumbsupunique.rb|db/migrate/20231023070913_thumbsupunique.rb|
