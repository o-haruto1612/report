## 感想

### 風澤宥吾

僕は新しいスタンプの作成を担当したが、アプリケーションを探索している中で気づいたことは、お気に入りボタンをクリックするという簡単な操作をするだけでも、アプリケーションが大規模であると関連する関数が多く、何重にも呼び出されているということだ。スタンプ機能の作成の部分で書いたような機能はクリックするとボタンが反応するというだけの簡単なものであり、探索している中でお気に入りボタンに関する機能は他にも、
- お気に入りの数をカウントする
- 投稿がお気に入り登録されたら投稿者に通知がいく

などがあり、それぞれの機能に対して多くの関数があり、アプリケーションの構造が複雑であると感じた。

実装していくなかで苦労した点は、お気に入りのスタンプを押した時の挙動を探索している際に関数や変数がどこで定義されているのかを見つけることである。そもそもファイル数が多く、関数名や変数名を検索してもヒットするものが多いことはもちろん、今回のアプリケーションを構成していたReactやRuby on railsは初めて使うフレームワークであり、Rubyについては言語から初めて見たので、構成や文法の表している意味を理解するのに時間がかかった。また、railsの性質で外部ファイルの内容を参照する際に、明示的にインポート文を記さなくても、自動で他のファイルの内容にアクセスできたり、以下のように
```
resource :favourite, only: :create
```
このように書くだけでFavouriteControllerクラスのcreate関数が呼び出すことができるなど呼び出しの部分を省略しても自動で適切な関数を呼び出せたりする性質があったため、どこのファイルのどの関数が呼び出されているかを探索することに時間がかかった。

スタンプ機能の説明をしているパートでは、順序立ててクリックされた時の挙動を示せたが、実際に探索している段階では、特にバックエンドの関数は断片的にしかわからず、全てのつながりを把握することに実験時間の大部分（２週間ほど）を取られた。なので、当たり前かもしれないが、探索する前に先にフレームワークの勉強をあらかじめ時間をとってしておくことにより、トータルで実装するのに時間がかなり短縮されると思うので、探索する前にフレームワークの予習は行っておくべきであると学習した。

