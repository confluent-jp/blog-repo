
## Reference
blog Repo：　https://github.com/confluent-jp/blog-repo

オリジナルRepo: https://github.com/wowchemy/starter-hugo-academic

サンプル: https://academic-demo.netlify.app/

## How
```bash
brew install hugo
git clone https://github.com/confluent-jp/blog-repo.git
cd blog-repo
hugo server
```
http://localhost:1313/confluent-jp/ にアクセス。

## User
```content/author``` 配下にユーザーIDによるフォルダを作成の上、```index.md```を配置。併せて画像ををavatar.png (avatar.jpg) という名前で設置するとポートレイトも足せます。 (是非登録してください。)

## Blog
```content/blog``` 配下にフォルダとして登録。
- フォルダ名は自由ながら、このままURLの一部となります。SEO的にもなるべく具体的な文言を含めてください。
- author - ご自身のuser IDを指定してください。
- date - 未来日付を入れておくとその日に公開されます。
- lastmod - 登録時の自動更新等は無いです。更新時は面倒ですが毎回```lastmod```の更新をお願いします。
- category - 厳密なリストではありませんが、以下の中から該当を記載してください。あまり増やすものでもありませんが、足していただいても結構です。
    - type - Blog or Announcement
    - area - Kafka Core, Kafka Connect, KSQL, Flink, Schema Registry, Confluent for Kubernetes, etc.
- tag - categoryよりは自由に足してください。
- ブログに関する画像を何か、featured.png (featured.jpg) という名前で同じフォルダに保存してください。プレビューとブログ本文と、それぞれ自動的にリサイズされたものが表示されます。
- 追加画像その他の利用方法はサンプルを参照してください。


