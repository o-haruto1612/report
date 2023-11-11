## 6. URL埋め込み機能  
ここではURL埋め込み機能の実装について説明していきます。
### 実装方針  
***
#### 1. リンク埋め込み機能について
リンク埋め込みとはそもそもどういう機能なのでしょうか。  
URLはそのまま表示すると以下のようになります。(Googleの初期ページ)  
https://www.google.co.jp/

このくらいの短さのリンクなら良いのですが、もっと長いリンクだったらどうでしょうか。(ChromeでMastodonと検索したときのページ)  
https://www.google.com/search?q=mastodon&rlz=1C5CHFA_enJP991JP1007&oq=mastodo&gs_lcrp=EgZjaHJvbWUqBwgAEAAYgAQyBwgAEAAYgAQyBggBEEUYOTIHCAIQABiABDIHCAMQABiABDIHCAQQABiABDIGCAUQRRg9MgYIBhBFGD0yBggHEEUYPagCALACAA&sourceid=chrome&ie=UTF-8

こんな長いリンクなんて出てきた際には文章全体がわかりにくくなってしまいます。  
そんな時Slackでは以下のようにリンクを文字に埋め込む表示方法ができます。
[このリンクにアクセス](https://www.google.com/search?q=mastodon&rlz=1C5CHFA_enJP991JP1007&oq=mastodo&gs_lcrp=EgZjaHJvbWUqBwgAEAAYgAQyBwgAEAAYgAQyBggBEEUYOTIHCAIQABiABDIHCAMQABiABDIHCAQQABiABDIGCAUQRRg9MgYIBhBFGD0yBggHEEUYPagCALACAA&sourceid=chrome&ie=UTF-8)  


それではMastodonの場合はどうでしょうか。  
長いURLをトゥート(投稿のこと)すると以下のように短縮された形で表示されます。  
![6-1.png](/image/6-1.png)  
Mastodonでのリンクの表示のされ方

ただしSlackのようにリンクを文字に埋め込むことはできません。この章ではこの機能をMastodonに実装していきます。

#### 2. 実装のために変更が必要な箇所

リンクを埋め込む機能の実装にはプログラムのどこを変更する必要があるでしょうか。  
以下が最初に浮かんだ変更必要箇所です。

1. __投稿するフォームのUI__  
リンクを埋め込むには、埋め込むリンクとリンクを埋め込むテキストの取得が必要です。そのためには投稿フォームのUIの変更が必要ですね。フロントエンドをいじることになりそうです。   

1. __UIで受け取ったデータの処理__  
投稿フォームのUIで受け取ったリンクとテキストのデータを、投稿を表示するUIに渡す必要があります。この部分はデータを管理することになるので、バックエンドに入ってくる可能性があります。  

1. __投稿を表示するUI__  
リンクとテキストを受け取っただけでは、表示できていないので、受け取ったリンクとテキストを実際の画面に反映する必要があります。こちらもUIの変更にあたるので、フロントエンドですね。 

<br>
次節以降でそれぞれの機能についてプログラムを手探りつつ修正を加えていきます！  


***
### 投稿するフォームのUI
***
#### 1. 方針
まずは投稿するフォームです！  
Mastodonでは投稿フォームは以下のように表示されています。  

![6-2.png](/image/6-2.png)  
Mastodonにおける投稿フォーム

ここでのテーマはこの投稿フォームにおいて指定した箇所に、リンクを埋め込めるように適切な情報を受け取ることです。今回は、リンクを埋め込みたいテキストをドラッグして選択すると、下にリンクを埋め込むテキストボックスが出てきて、そこにリンクをペーストすることで、リンクを埋め込めるようにしようと思います。  

まずは投稿フォームがどのファイルで書かれいているかを見つけなければいけません。今回は投稿フォームに書かれている、「今なにしてる」というキーワードに注目して、VSCode上の左に出てくる虫眼鏡マークから検索をかけてみます。  

`ja.json`の146行目の

    "compose_form.placeholder": "今なにしてる？",

がヒットしました。  
さらに"compose_form.placeholder"で検索すると、`compose_form.jsx`がヒットしました！`compose_form.jsx`では"ComposeForm"というクラスが定義されています。このクラスが投稿フォームを構成しているようですね。  
さらに"ComposeForm"の中を見ていくと、実際にテキストを打ち込む部分は"AutosuggestTextarea"というコンポーネントで定義されていることがわかります。  
今回はテキストエリアにおける選択を検知する必要があるので、"AutosuggestTextarea"を定義しているファイルをいじる必要があります。VSCodeではコンポーネントなどにカーソルを合わせて右クリックすると、それを定義している部分に飛ぶことができます。  
"AutosuggestTextarea"は`autosuggest_textarea.jsx`で定義されていることがわかります。  
次項では実際に`autosuggest_textarea.jsx`と`compose_form.jsx`に変更を加えていきます。




#### 2.コードの変更
それでは実際のコードの変更部分に入っていきます。  
今回いじる関数はどちらもjsxファイルで、Reactで書かれています。

#### 1. autosuggest_textarea.jsx  
"AutosuggestTextarea"では"onChange"や"onKeydown"などの、テキストエリア内での動きに対応する関数と、"textarea"という、実際のテキストエリアに対応するコンポーネントから構成されています。またその際に必要な情報はstateを使って管理しています。  
ここでは、テキストの選択と選択解除の検知をする関数"onSelect","onDeselect"、リンクを埋め込む上で必要な情報をstateに代入する関数"setLink"、そして選択状態に応じて画面に表示されるコンポーネント"LinkInsertArea"を新たに作ることで、リンクを埋め込めるようにします。  

まずはじめに、機能を実装する上で必要になるであろうstateを追加しておきます。  

<details><summary>stateの追加のコード</summary>

    state = {
    suggestionsHidden: true,
    focused: false,
    selectedSuggestion: 0,
    lastToken: null,
    tokenStart: 0,

    //ここから追加
    selectedText: "",
    selectionStart: 0,
    selectionEnd: 0,
    burryingtext: "",
    burryingtextStart: 0,
    burryingtextEnd: 0,
    burriedlink: "",
    //ここまで

    BoxsuggestionsHidden: true,
    Boxfocused: false,
    BoxselectedSuggestion: 0,
    BoxlastToken: null,
    BoxtokenStart: 0,
    };
</details>  
<br>
各ステートの役割は以下のようになります。  

- selectedText  
選択しているテキスト(選択しているテキストによってリアルタイムで変化)を保持する。
- selectionStart  
選択しているテキスト(選択しているテキストによってリアルタイムで変化)の最初の文字が全体のテキストの中での何番目に位置するかを保持する。
- selectionEnd    
選択しているテキスト(選択しているテキストによってリアルタイムで変化)の最後の文字が全体のテキストの中での何番目に位置するかを保持する。
- burryingtext  
リンクを埋め込む対象となるテキストを保持する。
- burryingtextStart  
リンクを埋め込む対象となるテキストの最初の文字が全体の投稿文の中での何番目に位置するかを保持する。
- burryingtextEnd  
リンクを埋め込む対象となるテキストの最後の文字が全体の投稿文の中での何番目に位置するかを保持する。
- burriedlink  
埋め込むリンクを保持する。

次に"onSelect", "onDeselect"関数を追加していきます。

<details><summary>"onSelect", "onDeselect"の追加のコード</summary>
    
    onPaste = (e) => {
    if (e.clipboardData && e.clipboardData.files.length === 1) {
      this.props.onPaste(e.clipboardData.files);
      e.preventDefault();
    }
    };

    //ここから追加
    onSelect = (e) => {
        const selectedText = window.getSelection().toString();
        this.setState({ selectedText: selectedText });
        this.setState({ selectionStart: e.target.selectionStart });
        this.setState({ selectionEnd: e.target.selectionEnd });
    };

    onDeselect = () => {//追加
        this.setState({ selectedText: "" });
    };
    //ここまで

    BoxonChange = (e) => {
</details>
<br>
各関数の役割は以下のようになります。  

- onSelect  
投稿文の中の一部のテキストを選択したときに、"selectedText", "selectionStart", "selectionEnd"の各ステートに値を代入する。
- onDeselect  
投稿文の中の一部のテキストの選択を解除したときに、"selectedText", のステートを初期化する。("selectionStart", "selectionEnd"については、更新する必要なし。実は"selectedText"を更新する必要もないが見栄えのため追加しておく。)  

次にsetLink関数を追加していきます。
<details><summary>"setLink"の追加のコード</summary>
    
    setLink = (text) => {
        this.setState({ burryingtext: this.state.selectedText });
        this.setState({ burriedlink: text });
        this.setState({ selectedText: "" });
        this.setState({ burryingtextStart: this.state.selectionStart});
        this.setState({ burryingtextEnd: this.state.selectionEnd});
    };
</details>
<br>

この関数は呼び出された時に、"textarea"にて選択されているテキストの情報を"burryingtext", "burryingtextStart", "burryingtextEnd"に代入し、"LinkInsertArea"内のテキストを"burriedlink"に代入することで、リンクを埋め込む上で必要な情報を全てstateで取得することができます。

次に"LinkInsertArea"コンポーネントを追加していきます。  
"LinkInsertArea"コンポーネントは`autosuggest_textarea.jsx`においてのみ使用するので、このファイルの一番最後の部分に単体で追加します。
<details><summary>"LinkInsertArea"の追加のコード</summary>
    
    //追加
    const LinkInsertArea = ({setLink}) => {
    LinkInsertArea.propTypes = {
        setLink: PropTypes.func.isRequired
    };
    const [text, setText] = useState("");
    const ontextchange = useCallback((e)=>{
        setText(e.target.value);
    }, []);
    const onClick = useCallback(()=>{
        setLink(text);
    }, [text]);
    return (
        <div>
        <label>
            <input
            type='text'
            value={text}
            onChange={ontextchange}
            placeholder={"リンクを入力"}
            />
        </label>
        <button onClick={onClick}>完了</button>
        </div>
    );
    };
</details>
<br>
またreactを使う必要があるので、二行目に以下のように追加しておきます。  
    
    import PropTypes from 'prop-types';
    import{useCallback, useState} from 'react';//追加
"LinkInsertArea"では、inputとしてtextを受け取り、onClickに応じて、"setLink"を呼び出すようにしています。これにより、inputから受け取ったリンクを"setLink"によりstateに渡すことができます。  
  
  

さて、ここまで来たらあとはreturnの中身において、"textarea"コンポーネントを修正し、さらにstateに応じて"LinkInsertArea"が表示されるようにすれば完了です。  
<details><summary>returnの中身の修正コード</summary>

    <div className='autosuggest-textarea'>
        <label>
        <span style={{ display: 'none' }}>{placeholder}</span>

        <Textarea
            ref={this.setTextarea}
            className='autosuggest-textarea__textarea'
            disabled={disabled}
            placeholder={placeholder}
            autoFocus={autoFocus}
            value={value}
            onChange={this.onChange}
            onKeyDown={this.onKeyDown}
            onKeyUp={onKeyUp}
            onFocus={this.onFocus}
            onBlur={this.onBlur}
            onPaste={this.onPaste}

            //ここから追加
            onSelect={this.onSelect} 
            onDeselect={this.onDeselect}
            //ここまで

            dir='auto'
            aria-autocomplete='list'
            lang={lang}
        />
        </label>
    </div>

    //ここから追加
    <div>
        {this.state.selectedText && ( // 選択テキストがある場合に表示
        <LinkInsertArea
            setLink={this.setLink}
        />
        )}
    </div>
    //ここまで
</details>
<br>
ここまでで、"burryingtext", "burryingtextStart", "burryingtextEnd", "burriedlink"の各ステートにリンクを埋め込み表示する上で必要な情報を代入することができました。  

#### 2. compose_form.jsx
`compose_form.jsx`にも少しだけ手を加える必要があります。  
今のままだと、"burryingtext", "burryingtextStart", "burryingtextEnd", "burriedlink"のステートの初期化が行えていないので、新しく投稿しようとすると、前の投稿におけるステートが残ったままになってしまいます。`autosuggest_textarea.jsx`では一つの投稿内で完結する操作しか行えないので、一つレイヤが上の`compose_form.jsx`にて初期化を行う必要があります。  

ここでは"ComposeForm"クラスの中の、投稿を実際にサブミットする操作を管理する関数、"handleSubmit"に変更を加えていきます。
<details><summary>"handleSubmit"の変更のコード</summary>

    //ここから追加
    //ここでstateのリセット
    this.autosuggestTextarea.setState({ burryingtext: "" });
    this.autosuggestTextarea.setState({ burriedlink: "" });
    this.autosuggestTextarea.setState({ burryingtextStart: 0});
    this.autosuggestTextarea.setState({ burryingtextEnd: 0});
    //ここまで追加

    if (e) {
      e.preventDefault();
    }
  };
</details>
<br>
これにより、投稿した後に各stateを初期化することができました。  

以上により、`autosuggest_textarea.jsx`と`compose_form.jsx`が適切に変更され、以下のように投稿フォームからリンクと埋め込むテキストの情報を受け取ることができました!!  
![6-3.png](/image/6-3.png)  
Mastodonにおける投稿フォーム

***

### UIで受け取ったデータの処理
***
#### 1. 方針
次は投稿を表示する部分です！  
前節までで、リンクと埋め込むテキストの情報を得ることができました。ここでのテーマは受け取った情報を投稿を表示するUIにどのように伝達するかです。  

伝達する方法として思いついたものは二つほどありました。  
一つ目は、新しく受け取ったリンクや埋め込むテキストの情報を格納するデータベースを作成し、そこにデータを保持する方法です。  
二つ目は、受け取ったテキストと埋め込むリンクをMarkdown形式に変換して、投稿文の情報の中にリンク埋め込みを含めた全ての情報を入れ込んでしまう方法です。例えば以下の文の"ここ"にリンクを埋め込む場合は、stateに保持する情報を使って、上の元の投稿文を下のMarkdown式に変更します。  
    
    元の投稿文：
    "ここからgoogleのトップページに飛べます。"  

    Markdown形式変更後の投稿文：
    "[ここ](https://www.google.co.jp/)からgoogleのトップページに飛べます。"

一つ目の方法に関しては、データベースなどプログラムの深部まで改変を行う必要があり、それに付随してよりたくさんのファイルに変更を加える必要があります。それに対し二つ目の方法に関しては、実質的には投稿文に手を加えているだけなので、データベースなどはなにも変わりません。よって今回は二つ目の選択肢を採用することにしました。  

具体的な手法としては前節でも少し手を加えた`compose_form.jsx`に変更を加えていきます。
#### 2.コードの変更
#### compose_form.jsx
今回は`compose_form.jsx`のみの変更で良さそうです。  
ここでは前節でも変更を加えた、"handleSubmit"関数において、投稿文をMarkdown式で書き直してfinaltextに代入し、submitするようにします。
<details><summary>"handleSubmit"の変更のコード</summary>
    
    handleSubmit = (e) => {
        if (this.props.text !== this.autosuggestTextarea.textarea.value ) {
        // Something changed the text inside the textarea (e.g. browser extensions like Grammarly)
        // Update the state to match the current text
        this.props.onChange(this.autosuggestTextarea.textarea.value);
        }

        if (!this.canSubmit()) {
        return;
        }

        //ここから変更
        const finaltext =
        this.autosuggestTextarea.textarea.value.slice(0, this.autosuggestTextarea.state.burryingtextStart)
        + '[' +  this.autosuggestTextarea.state.burryingtext + ']'
        + '(' + this.autosuggestTextarea.state.burriedlink + ')'
        + this.autosuggestTextarea.textarea.value.slice(this.autosuggestTextarea.state.burryingtextEnd);
        //ここでtextの内容をつなげてmarkdown形式にして送る

        this.props.onChange(finaltext);
        this.props.onSubmit(this.context.router ? this.context.router.history : null);
        //ここまで

        //ここから前節の追加分
        //ここでstateのリセット
        this.autosuggestTextarea.setState({ burryingtext: "" });
        this.autosuggestTextarea.setState({ burriedlink: "" });
        this.autosuggestTextarea.setState({ burryingtextStart: 0});
        this.autosuggestTextarea.setState({ burryingtextEnd: 0});
        //ここまで前節の追加分

        if (e) {
        e.preventDefault();
        }
    };
</details>
<br>
以上で投稿フォームで受け取った情報をMarkdown形式として投稿文に落とし込むことができました。  

***

### 投稿を表示するUI
***
#### 1. 方針
最後に投稿を表示する部分です！  
前節までで、リンクと埋め込むテキストの情報をMarkdown形式で得ることができました。ここでは、実際にリンクが埋め込まれた状態で、投稿が表示されるようにしたいと思います。  

Mastodonでは通常、長いリンクをトゥートした場合、短縮して表示されます。ということはURLを短縮して表示するための関数があるはずですね。  
今回もVSCodeの検索で色々調べてみます。いくつかのワードを試した後、"shortened"と検索すると`text_formatter.rb`内に"shortened_link"という関数が定義されていることがわかりました。この関数内でURLの短縮が行われているようです。  
さらに`text_formatter.rb`を調べると、このファイルでは投稿文のテキストからリンクやハッシュタグなどを検出して、実際に表示するhtml形式に変換していることがわかりました。このファイルに主に変更を加えていけば目的を達成できそうです！！  

さて、`text_formatter.rb`内ではurlのハイパーリンク化に関わっていそうな関数を以下のように三つ発見しました。  

- to_s  
urlやhashtagなどの有無に応じて、link_to_urlなどのhtmlの変換を加える関数を呼び出し、投稿文に変更を加え、最終的にhtml形式の投稿文を完成させる。
- link_to_url  
オブジェクトを引数にとる。渡されたオブジェクトのurl部分(すでに別の関数で正規表現により抽出されている)をshortened_linkに渡す。
- shortened_link  
urlを引数にとる。受け取ったurlの長さに応じて、urlを短縮表示するか判断し、適切にhtml形式に落とし込む。  

これらの関数に変更を加えていけば、うまく投稿文を整形できそうです。  
<br>
今回は大まかには以下の作戦でMarkdown形式を変更していきます。  
1. "link_to_url"において正規表現でtextとurlを抽出し、textとurlの両方を"shortened_link"に渡す。  
1. "shortened_link"にtextとurlを渡すことでtextにurlを埋め込んだ状態のhtmlを生成する。  
1. "to_s"において正規表現により、text以外の部分を削除する。  

<br>
イメージとしては以下のように変更が加えられていきます。  

1. 最初のMarkdown形式：
        
        "[ここ](https://www.google.co.jp/)からgoogleのトップページに飛べます。"
    ※この記法でないとハイパーリンク化のエスケープができないのでご了承ください。
1. "ここ"の部分がハイパーリンク化した状態：  
"[[ここ](https://www.google.co.jp/)](https://www.google.co.jp/)からgoogleのトップページに飛べます。" 
1. "ここ"の部分がハイパーリンク化した状態で他を削除：  
"[ここ](https://www.google.co.jp/)からgoogleのトップページに飛べます。"


#### 2.コードの変更
#### 1. text_formatter.jsx
それでは実際にコードに変更を加えていきます。
##### a. shortened_link  
textに'0'が渡される場合は通常のリンク短縮を、それ以外の場合はtextにurlを埋め込んだ形にすることにしました。  
<details><summary>"shortened_link"の変更のコード</summary>
    
        def shortened_link(url, text, rel_me: false)
            url = Addressable::URI.parse(url).to_s
            rel = rel_me ? (DEFAULT_REL + %w(me)) : DEFAULT_REL

            prefix = url.match(URL_PREFIX_REGEX).to_s
            if text == '0' 
                display_url = url[prefix.length, 30]
                suffix      = url[prefix.length + 30..]
                cutoff      = url[prefix.length..].length > 30
            else
                display_url = text
                suffix      = url[prefix.length + 30..]
                cutoff      = url[prefix.length].length > 30
            end

            <<~HTML.squish.html_safe # rubocop:disable Rails/OutputSafety
                <a href="#{h(url)}" target="_blank" rel="#{rel.join(' ')}" translate="no"><span class="invisible">#{h(prefix)}</span><span class="#{cutoff ? 'ellipsis' : ''}">#{h(display_url)}</span><span class="invisible">#{h(suffix)}</span></a>
            HTML
</details>

##### b. link_to_url  
正規表現で抽出できた場合は抽出したtextをurlと一緒にshortened_linkに渡し、それ以外の場合は'0'をurlと一緒にshortened_linkに渡すことにしました。  
<details><summary>"link_to_url"の変更後のコード</summary>
        
        def link_to_url(entity)
            pattern = /\[([^\]]+)\]\(([^)]+)\)/
            matches = text.scan(pattern)
            if matches.empty?
            TextFormatter.shortened_link(entity[:url], '0', rel_me: with_rel_me?)
            else
            word = matches[0][0]
            TextFormatter.shortened_link(entity[:url], word, rel_me: with_rel_me?)
            end
        end
</details>

##### c. to_s  
正規表現で[](url)の部分を削除するコードを追加しました。  
<details><summary>"to_s"の変更後のコード</summary>
        
        def to_s
            return ''.html_safe if text.blank?

            html = rewrite do |entity|
            if entity[:url]
                link_to_url(entity)
            elsif entity[:hashtag]
                link_to_hashtag(entity)
            elsif entity[:screen_name]
                link_to_mention(entity)
            end
            end
            html = html.gsub(/\[(.*?)\]\((.*?)\)/, '\2')

            html = simple_format(html, {}, sanitize: false).delete("\n") if multiline?

            html.html_safe # rubocop:disable Rails/OutputSafety
        end
</details>
<br>

#### 2. formatting_helper.jsx  
`formatting_helper`内の関数である"account_field_value_format"においても、"shortened_link"が呼び出されているので、引数を変更しておきます。ここではリンクの埋め込みは関係なさそうなので、'0'を引数に入れておきます。  
<details><summary>"account_field_value_format"の変更後のコード</summary>

    def account_field_value_format(field, with_rel_me: true)
        if field.verified? && !field.account.local?
        TextFormatter.shortened_link(field.value_for_verification, '0')
        else
        html_aware_format(field.value, field.account.local?, with_rel_me: with_rel_me, with_domains: true, multiline: false)
        end
    end
</details>
<br>

***

### 完成！！
***
以上により実装できました！！  
デモ動画を以下に載せておくので、ぜひご覧ください！！


