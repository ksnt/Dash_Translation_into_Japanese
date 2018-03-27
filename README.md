#  このリポジトリについて

Dashの英語チュートリアルを日本語に翻訳するプロジェクトです．  
このリポジトリは公式ドキュメントにマージされることにより破棄される可能性があります．当面は一時的なリポジトリとなります．  

📢このリポジトリに関する議論は以下のリンク先で行われています！

[英語チュートリアルを日本語に翻訳してもいいかな？](https://community.plot.ly/t/can-i-translate-english-tutorial-into-japanese-one/8859?u=ksnt)

ご協力いただける方歓迎です！  
お気軽にご連絡ください！  

# Dashチュートリアル  

## 第1部 インストール

## [第2部 Dash レイアウト](https://github.com/ksnt/Dash_Translation_into_Japanese/blob/master/dash_tutorial_jap_chap1.md)

Dashは`layout`を使ってあなたのアプリケーションの見え方を指定します。これは、一連の宣言的なDashコンポーネントによって構成されています。

## 第3部 コールバック(`Callback`)の基礎

DashアプリケーションはDashコールバック(Dash Callbacks)を通じて対話的な機能を実現しています。コールバックというのは、入力コンポーネントの性質が変わったときにはいつも自動的に呼ばれるPythonの関数のことです。コールバックは複数のコールバックを連ねることが出来ます。また、一つのアプリケーションに閉じない複数のアップデートを引き起こすようなユーザーインターフェースにおいては、一度限りのアップデートを行います。


## 第4部 状態(`State`)を持ったコールバック

基本的なコールバックは値が変化したときにいつも呼びだされます。インプットが変わったときはつねに特別な値を反映させるためにDash StateをDash Inputsと一緒に使ってください。  


## 第5部 対話的なグラフ作成とクロスフィルタリング

あなたがカーソルを合わせたり（hover）、クリックしたり、チャート上の点を選んだときはいつでも対話的な操作が行えるように、Dashグラフコンポーネント(the Dash Graph component)を使いましょう。


## 第6部 コールバック間でのデータ共有

グローバル変数はあなたのDashアプリケーションを破壊してしまう危険があります。しかし心配はいりません。コールバック間でデータを共有するための他の方法が存在します。この章の知識は重いデータ処理タスクを実行させたいときや大きなデータを操作するコールバックを用いるときに役立つことになるでしょう。


# 参考

[公式ドキュメント](https://github.com/plotly/dash-docs)


# Translation of Dash English tutorial into Japanese

I am working on translation English Dash tutorial into Japanese one. 
This repository will be aborted when marged into official documents. This repository is temporary.    

📢 Please see the link below if you want to see discussion on this repository!  

[Can I translate English tutorial into Japanese one?](https://community.plot.ly/t/can-i-translate-english-tutorial-into-japanese-one/8859?u=ksnt)

Any collaborators are welcome!  
Feel free to contact me here!  

# Reference

[Original Documents](https://github.com/plotly/dash-docs)
