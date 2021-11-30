# Embulk を使ったCSVファイルの転送（Dockerでの開発環境版）


## Embulkとは

>Embulkは、さまざまなストレージ、データベース、NoSQL、クラウドサービス間>のデータ転送を支援する並列バルクデータローダーです。
>
>Embulkは、関数を追加するためのプラグインをサポートしています。プラグインを共有して、カスタムスクリプトを読みやすく、保守しやすく、再利用できるようにすることができます。

([公式Github](https://github.com/embulk/embulk)より　日本語訳)

[Embulk 公式ドキュメント](https://www.embulk.org/docs/)

## できることとメリット
- 機能面
  - CSVファイルの内容をDBに入れることができる。
  - エラーが起きた時はデータ転送を完全に中断するため安全にデータ挿入ができる
- 開発環境面
  - 同一なテストデータを素早く使いまわせる
  - 拡張プラグインを導入することで豊富な機能を拡張性高く使える
    - [Filterプラグイン](https://github.com/sonots/embulk-filter-column)
    - [外部連携プラグイン](https://github.com/embulk/embulk-output-bigquery)
- 補足
  - Treasure Dataは外部連携に Embulk を使っている
    - Shopifyとの連携では独自にプラグインを作成しているみたい
  - MAGNETでも今後利用してもいいのかも
    - WebアプリケーションのロジックとETLのロジックを切り離せる
    - ETLを高品質に行える

## 使い方

1. CSVファイル(tsvでも圧縮ファイルでも可)を/app配下に置く
2. CSVファイルの自動スキーマ検知のためのファイルを作成する
    - ファイルの拡張子は.yml.liquid [.liquidファイルとは?](https://obel.hatenablog.jp/entry/20171228/1514433459)  
    - コードの構成は大きく in と out の2つに分かれる
    - csvやtsv等のファイルの ローカルでのread/write に関してはembulkの標準機能で完結できる
    - このDocker環境下にはmysql-output-pluginをインストールして、outではmysqlにデータを吐き出している

    コード例 (customer_seed.yml.liquid)
    ```yml :customer_seed.yml.liquid
    in:
      type: file
      path_prefix: "./dd_customer_table.csv"
    out:
      type: mysql
      host: db
      user: {{ env.MYSQL_USER }}
      password: {{ env.MYSQL_PASSWORD }}
      database: db
      table: customer_table
      mode: replace
    ```
3. 自動スキーマ検知とデータロード用ファイルの自動作成
    - Embulkのコマンドを使ってデータロード用のファイルを自動作成する

    ```shell
    # 一旦ファイル確認
    $ ls
    $ customer_seed.yml.liquid
    
    $ embulk guess ./customer_seed.yml.liquid -o customer_load.yml.liquid
    ``` 
    - ファイルの内容はこんな感じ。環境変数が展開されてしまうが、なにか防止策があるかもしれない。
    ```yml
    in:
      type: file
      path_prefix: ./dd_customer_table.csv
      parser:
        charset: UTF-8
        newline: LF
        type: csv
        delimiter: ','
        quote: '"'
        escape: '"'
        trim_if_not_quoted: false
        skip_header_lines: 1
        allow_extra_columns: false
        allow_optional_columns: false
        columns:
          - {name: customer_id, type: string}
          - {name: email, type: string}
          - {name: phone_number, type: string}
          - {name: last_name, type: string}
          - {name: first_name, type: string}
          - {name: gender, type: string}
          - {name: age, type: long}
          - {name: birthday, type: timestamp, format: '%Y/%m/%d'}
          - {name: prefectures, type: string}
          - {name: card_issuance_date, type: timestamp, format: '%Y/%m/%d'}
          - {name: last_purchase_date, type: timestamp, format: '%Y/%m/%d'}
          - {name: line_user_id, type: string}
    out: 
      type: mysql
      host: db
      user: user     # <-これは {{ env.MYSQL_USER }}に変更する
      password: pass # <-これは {{ env.MYSQL_PASSWORD }}に変更する
      database: db
      table: customer_table
      mode: replace
    ```
    - timestamp型だと1970年~2040年くらいしか使用できないため、DATE型としてMYSQLに送り込むために、次のようにする

    ```yml
    out:
      ... 省略...
      mode: replace
      column_options:
        birthday: {type: date, value_type: string}
        card_issuance_date: {type: date, value_type: string}
        last_purchase_date: {type: date, value_type: string}
    ```
4. データ転送
    - 次のコマンドを打ち実行する
    ```shell
    $ embulk run ./customer_load.yml.liquid
    ```

    - 実行がうまくいけば MYSQL にテーブルが作成され、データがはいいていることが確認できる。


  以上。
