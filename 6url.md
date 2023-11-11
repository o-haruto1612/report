## 6. URL埋め込み機能

### 実装方針
#### 1. リンク埋め込み機能について
リンク埋め込みとはそもそもどういう機能なのでしょうか。  

URLはそのまま表示すると以下のようになります。(Googleの初期ページ)  
https://www.google.co.jp/

このくらいの短さのリンクなら良いのですが、もっと長いリンクだったらどうでしょうか。(ChromeでMastodonと検索したときのページ)  

https://www.google.com/search?q=mastodon&rlz=1C5CHFA_enJP991JP1007&oq=mastodo&gs_lcrp=EgZjaHJvbWUqBwgAEAAYgAQyBwgAEAAYgAQyBggBEEUYOTIHCAIQABiABDIHCAMQABiABDIHCAQQABiABDIGCAUQRRg9MgYIBhBFGD0yBggHEEUYPagCALACAA&sourceid=chrome&ie=UTF-8

こんな長いリンクなんて出てきた際には文章全体がわかりにくくなってしまいます。  
そんな時Slackでは以下のようにリンクを文字に埋め込む表示方法ができます。

[このリンクにアクセス](https://www.google.com/search?q=mastodon&rlz=1C5CHFA_enJP991JP1007&oq=mastodo&gs_lcrp=EgZjaHJvbWUqBwgAEAAYgAQyBwgAEAAYgAQyBggBEEUYOTIHCAIQABiABDIHCAMQABiABDIHCAQQABiABDIGCAUQRRg9MgYIBhBFGD0yBggHEEUYPagCALACAA&sourceid=chrome&ie=UTF-8)  

Slackでの実際の画面は以下のようになります。  


<!--画像挿入-->
【Slackの実際の画面の様子】  




それではMastodonの場合はどうでしょうか。  
長いURLをトゥート(投稿のこと)すると以下のようになります。  
<!--画像挿入-->

ただしSlackのようにリンクを文字に埋め込むことはできません。この章ではこの機能をMastodonに実装していきます。

#### 2. 実装のために変更が必要な箇所

リンクを埋め込む機能の実装にはプログラムのどこを変更する必要があるでしょうか。  
以下が最初に浮かんだ変更必要箇所です。

1. __投稿するフォームのUI__  
リンクを埋め込むには、埋め込むリンクとリンクを埋め込むテキストの取得が必要です。そのためには投稿フォームのUIの変更が必要ですね。フロントエンドをいじることになりそうです。  

1. __投稿を表示するUI__  
リンクとテキストを受け取っただけでは、表示できていないので、受け取ったリンクとテキストを実際の画面に反映する必要があります。こちらもUIの変更にあたるので、フロントエンドですね。  

1. __UIで受け取ったデータの処理__  
投稿フォームのUIで受け取ったリンクとテキストのデータを、投稿を表示するUIに渡す必要があります。この部分はデータを管理することになるので、バックエンドに入ってくる可能性があります。  

次節以降でそれぞれの機能についてプログラムを手探りつつ修正を加えていきます！

***

### 投稿するフォームのUIの変更
#### 1. 方針
まずは投稿するフォームです！  
Mastodonでは投稿フォームは以下のように表示されています。  

<!--画像挿入-->

ここでのテーマはこの投稿フォームにおいて指定した箇所に、リンクを埋め込めるように適切な情報を受け取ることです。今回は、リンクを埋め込みたいテキストをドラッグして選択すると、下にリンクを埋め込むテキストボックスが出てきて、そこにリンクをペーストすることで、リンクを埋め込めるようにしようと思います。  

まずは投稿フォームがどのファイルで書かれいているかを見つけなければいけません。今回は投稿フォームに書かれている、「今なにしてる」というキーワードに注目して、VSCode上の左に出てくる虫眼鏡マークから検索をかけてみます。  

`ja.json`の146行目の

    "compose_form.placeholder": "今なにしてる？",

がヒットしました。  

さらに"compose_form.placeholder"で検索すると、`compose_form.jsx`がヒットしました！`compose_form.jsx`では"ComposeForm"というクラスが定義されています。このクラスが投稿フォームを構成しているようですね。  

さらに"ComposeForm"の中を見ていくと、実際にテキストを打ち込む部分は"AutosuggestTextarea"というコンポーネントで定義されていることがわかります。  

今回はテキストエリアにおける選択を検知する必要があるので、"AutosuggestTextarea"を定義しているファイルをいじる必要があります。VSCodeではコンポーネントなどにカーソルを合わせて右クリックすると、それを定義している部分に飛ぶことができます。  
"AutosuggestTextarea"は`autosuggest_textarea.jsx`で定義されていることがわかります。  

次節では実際に`autosuggest_textarea.jsx`と`compose_form.jsx`に変更を加えていきます。




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

以上により、`autosuggest_textarea.jsx`と`compose_form.jsx`が適切に変更され、以下のように投稿フォームからリンクと埋め込むテキストの情報を受け取ることができました。
<!--画像挿入-->

***

### 投稿を表示するUIの変更
#### 1. 方針
ここでのテーマ
やう必要のあること
この部分→この関数
この部分→この関数
#### 2.コードの変更
1. text_formatter.jsx
1. formatting_helper.jsx
***

### UIで受け取ったデータの処理の変更
#### 1. 方針
ここでのテーマ
やう必要のあること
この部分→この関数
この部分→この関数
#### 2.コードの変更
1. compose_form.jsx
1. text_formatter.rb
1. formatting_helper.rb
***

###



## 使えそうな記法

<details>
<summary>
サンプルコード
</summary>

(上に空行が必要)

```rb
puts 'Hello, World'
```
</details>

![Qiita](https://qiita-image-store.s3.amazonaws.com/0/45617/015bd058-7ea0-e6a5-b9cb-36a4fb38e59c.png "Qiita")

<img width="50" src="https://qiita-image-store.s3.amazonaws.com/0/45617/015bd058-7ea0-e6a5-b9cb-36a4fb38e59c.png">

<!-- コメントアウトしたい内容 -->