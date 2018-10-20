#  このリポジトリについて

**最終更新日: OCt 20, 2018**

Plotly社が開発しているPythonフレームワークであるDashの英語チュートリアルを日本語に翻訳するプロジェクトです．  
このリポジトリは公式ドキュメントにマージされることにより破棄される可能性があります．当面は一時的なリポジトリとなります．  
全てのアップデートに対応しているわけではないので，最新版のドキュメントは公式サイトを参考にしてください．  

📢このリポジトリに関する議論は以下のリンク先で行われています！

[英語チュートリアルを日本語に翻訳してもいいかな？](https://community.plot.ly/t/can-i-translate-english-tutorial-into-japanese-one/8859?u=ksnt)

ご協力いただける方歓迎です！  
お気軽にご連絡ください！  

# Dashユーザーガイド（日本語版）

## Dashって何？

[はじめに](https://github.com/ksnt/Dash_Translation_into_Japanese/blob/master/introcution.md)   
Dashについてのちょっとした記事とすべての始まりだったPlotconでのトークへのリンク  

[Announcement Essay](https://medium.com/@plotlygraphs/introducing-dash-5ecf7191b503)  
Dashについての追加記事．Dashのアーキテクチャとプロジェクトの背後にあるモチベーションに関する議論  

[Dashアプリケーションギャラリー](https://dash.plot.ly/gallery)  
Dashを使って何ができるのかをちらっと一瞥  

[Dashクラブ](https://plot.us12.list-manage.com/subscribe?u=28d7f8f0685d044fb51f0d4ee&id=0c1cb734d7)  
Dashの作者chriddypからの隔週ニュースレター

## Dashチュートリアル  

[第1部 インストール](https://github.com/ksnt/Dash_Translation_into_Japanese/blob/master/dash_tutorial_jap_chap0.md)

[第2部 Dash レイアウト](https://github.com/ksnt/Dash_Translation_into_Japanese/blob/master/dash_tutorial_jap_chap1.md)
Dashは`layout`を使ってあなたのアプリケーションの見え方を記述します。これは、一連の宣言的なDashコンポーネントによって構成されています。  

[第3部 コールバックの基礎](https://github.com/ksnt/Dash_Translation_into_Japanese/blob/master/dash_tutorial_jap_chap2.md)  
DashアプリケーションはDashコールバック(Dash Callbacks)を通じて対話的な機能を実現しています。コールバック(`Callback`)というのは、入力コンポーネントの性質が変わったときにはいつも自動的に呼ばれるPythonの関数のことです。コールバックは複数のコールバックを連ねることが出来ます。また、1つのアプリケーションに閉じない複数のアップデートを引き起こすようなユーザーインターフェースにおいては、1度限りのアップデートを行います。

[第4部 状態(State)を持ったコールバック](https://github.com/ksnt/Dash_Translation_into_Japanese/blob/master/dash_tutorial_jap_chap3.md)  
基本的なコールバックは値が変化したときにいつも呼びだされます。インプットが変わったときにはいつも追加の値を反映させるためにDash StateをDash Inputsと一緒に使ってください。  


[第5部 対話的なグラフ作成とクロスフィルタリング](https://github.com/ksnt/Dash_Translation_into_Japanese/blob/master/dash_tutorial_jap_chap4.md)  
あなたがカーソルを合わせたり（hover）、クリックしたり、チャート上の点を選んだときはいつでも対話的な操作が行えるように、Dashグラフコンポーネント(the Dash Graph component)を使いましょう。


[第6部 コールバック間でのデータ共有](https://github.com/ksnt/Dash_Translation_into_Japanese/blob/master/dash_tutorial_jap_chap5.md)  
グローバル変数はあなたのDashアプリケーションを破壊してしまう危険があります。しかし心配はいりません。コールバック間でデータを共有するために他の方法が存在します。この章の知識は重いデータ処理タスクを実行させたいときや大きなデータを操作するコールバックを用いるときに役立つことになるでしょう。

## コンポーネントライブラリ

[Dashコアコンポーネント](https://dash.plot.ly/dash-core-components)  
Dashコアコンポーネントライブラリにはスライダー、グラフ、ドロップダウン、テーブル、その他といった一連のより抽象度の高いコンポーネントが含まれています。

[Dash HTMLコンポーネント](https://dash.plot.ly/dash-html-components)  
DashはユーザーフレンドリーなPythonクラスとして利用可能なHTMLタグの全てを提供しています。この賞ではこれがどのように機能しているのかということとDashのHTMLコンポーネントと標準的なHTMLの間にあるいくつかの重要な違いについて説明します。

[Dash DAQコンポーネント](https://www.dashdaq.io/)
$840/開発者/年  
データ活用ユーザーインターフェースのための美しいスタイルと特別なコンポーネント  

## 自作コンポーネントを作る

[Python開発者のためのReact](https://dash.plot.ly/react-for-python-developers)  
Python開発者のためのReactおよびJavaScriptチュートリアル  

[自分でコンポーネントをつくる](https://dash.plot.ly/plugins)  
DashコンポーネントはReact.jsで作られています。DashはPythonで書かれたこれらのコンポーネントに対してDashに適合したインターフェースを生成するReactからDashへのツールチェーンを提供します。

## 発展的な内容

[パフォーマンス](https://dash.plot.ly/performance)  
Dashアプリケーションを高速化するためには2つの方法があります。一つはキャッシングでもう一つはWebGLチャートを使うことです。

[逐次更新](https://dash.plot.ly/live-updates)  
ページロードの際あるいは前もって決めた間隔(例:30秒毎)でアプリケーションを更新させてみましょう。

[CSSとJSを追加してPage-Loadテンプレートをオーバーライドする](https://dash.plot.ly/external-resources)  
Dashのバージョン0.22における新機能です。`assets`ディレクトリを使ってあなたのアプリケーションにカスタムしたCSSやJSを追加する方法について学びます。また、カスタムされたメタタグを追加したりページのタイトルをカスタマイズしたりするために、Dashがページロードの際に提供するHTMLテンプレートをカスタマイズする方法も学びます。

[URLルーティングとマルチプルアプリケーション](https://dash.plot.ly/urls)  
Dashは"Single Page App(SPA)"デザインパターンを使って高速で複数ページのアプリケーションを簡単に作れるようにするための2つのコンポーネント(`dcc.Link`と`dcc.Location`)を提供します。

## 参考

[公式ドキュメント](https://github.com/plotly/dash-docs)


# Translation of Dash English tutorial into Japanese

I am working on translation English Dash tutorial into Japanese one. Dash is a Python framework developed by Plotly.  
This repository will be aborted when marged into official documents. This repository is temporary.    

📢 Please see the link below if you want to see discussion on this repository!  

[Can I translate English tutorial into Japanese one?](https://community.plot.ly/t/can-i-translate-english-tutorial-into-japanese-one/8859?u=ksnt)

Any collaborators are welcome!  
Feel free to contact me here!  

# Reference

[Original Documents](https://github.com/plotly/dash-docs)
