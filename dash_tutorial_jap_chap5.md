# コールバック(Callbacks)間で状態を共有する

Dashのコアとなっている原理のうちの一つをこの入門ガイド内で説明します。それは、**Dash Callbacksは決してそのスコープ範囲外の変数から変更されてはいけない**ということです。この章ではなぜそれがいけないのかということを説明し、コールバック間での状態共有のための代替的なパターンをいくつか提示します。

## なぜ状態を共有するのか？

いくつかのアプリケーションでは複数のコールバックをもつことができるかもしれません。そうしたコールバックはSQLクエリを発行したりシミュレーションを走らせたりデータをダウンロードしたりといったような高い負荷のかかるタスクに依存したものでしょう。

各々のコールバックに同様の高負荷なタスクを走らせる代わりに、一つのコールバック上でタスクを走らせてから残りのコールバックにその結果を共有するさせることができます。


## なぜグローバル変数があなたのアプリケーションを壊すのか？

Dashはマルチユーザー環境で動くようにデザインされています。マルチユーザー環境というのは複数の人々が同時にアプリケーションを見たり独立のセッションを持ったりするような環境のことです。

もしあなたのアプリケーションが変更されたグローバル変数を使っていたら、その変数をある一人のユーザーのセッションが別のユーザーに影響を与えてしまうような変数に設定することができてしまいます。

Dashはまた複数のpythonのworker(実際にジョブを実行するプロセス)とともに走らせることが出来るようにデザインされています。このことによってコールバックを並行に実行することができます。これは通常gnicornのシンタックスを用いて次のように行うことができます。

$ gunicorn --workers 4 --threads 2 app:server

Dashアプリケーションが複数のworkersにわたって動くとき、メモリが共有されることはありません。これは、もし一つのコールバック内のグローバル変数を修正したらその修正は残りのworkerに適用されることはないということです。

スコープ外のデータを修正するコールバックを持つアプリケーションのスケッチがこちらになります。この種類のパターンは上記に述べられたような理由から*動作を保証しません*。

```python
df = pd.DataFrame({
    'a': [1, 2, 3],
    'b': [4, 1, 4],
    'c': ['x', 'y', 'z'],
})

app.layout = html.Div([
    dcc.Dropdown(
        id='dropdown',
        options=[{'label': i, 'value': i} for i in df['c'].unique()],
        value='a'
    ),
    html.Div(id='output'),
])

@app.callback(Output('output', 'children'),
              [Input('dropdown', 'value')])
def update_output_1(value):
    # Here, `df` is an example of a variable that is
    # "outside the scope of this function".
    # *It is not safe to modify or reassign this variable
    #  inside this callback.*
    global df = df[df['c'] == value]  # do not do this, this is not safe!
    return len(df)
```

この例を正しく直すためには、単純にフィルターをコールバック内の新しい変数に再度指定する、あるいはこのガイドの次の部分で述べられることになるストラテジーのうちの一つに従ってください。

```python
df = pd.DataFrame({
    'a': [1, 2, 3],
    'b': [4, 1, 4],
    'c': ['x', 'y', 'z'],
})

app.layout = html.Div([
    dcc.Dropdown(
        id='dropdown',
        options=[{'label': i, 'value': i} for i in df['c'].unique()],
        value='a'
    ),
    html.Div(id='output'),
])

@app.callback(Output('output', 'children'),
              [Input('dropdown', 'value')])
def update_output_1(value):
    # Safely reassign the filter to a new variable
    filtered_df = df[df['c'] == value]
    return len(filtered_df)

```

## コールバック間でデータを共有する

複数のpythonプロセスに亘ってデータを安全に共有するためには各々のプロセスがアクセス可能などこかにデータを格納しておく必要があります。データを格納するための場所は3つあります：

1 - ユーザーのブラウザのセッション内

2 - ディスク上 (例えば、ファイルや新しいデータベース上)

3 - 共有メモリ空間（例えば、Redisと一緒に使う）

次の3つの例はこれらのアプローチを示す例となっています。

## 例1 - 隠されたDiv要素内にデータを格納する


データをユーザーのブラウザのセッション内に保存する:  

- https://community.plot.ly/t/sharing-a-dataframe-between-plots/6173 で説明されている方法でDashのフロントエンドストア(front-end store)の部分としてデータを保存することで実装される  

- データは保存と移行のためJSONのような文字列に変換されなければいけない  

- この方法でキャッシュされるデータはユーザーの現在のセッションないでのみ利用できる

>　もしあたらしいブラウザで開いたら、アプリケーションのコールバックはつねにデータを計算する(compute)でしょう。データはそのセッションの範囲内のコールバック間でのみキャッシュされ移送されることになります。

> それ自体は、キャッシングをともなったものとは異なり、この方法はアプリケーションのメモリの足跡を増やすことはありません。  

> ネットワーク移送におけるコストはあるでしょう。もしコールバック間で10MBのデータを共有するのであれば、そのデータは各コールバック間でネットワークを越えて転送されるでしょう。

> もしネットワークコストが高すぎるのであれば、前もって全体(のコスト)を計算してそれらを転送してください。アプリケーションはおそらく10MBのデータを表示せず、その全体の一部を表示するだけでしょう。 

この例はいかにして一つのコールバック内で負荷の高いデータ処理のステップをやり遂げ、出力をJSONに変換し、それを他のオールバックへ入力として与えることができるかということを示しています。この例では標準的なDashのコールバックを使い、アプリケーション内の隠されたdiv要素内にJSON化されたデータを格納しています。

```python

global_df = pd.read_csv('...')
app.layout = html.Div([
    dcc.Graph(id='graph'),
    html.Table(id='table'),
    dcc.Dropdown(id='dropdown'),

    # Hidden div inside the app that stores the intermediate value
    html.Div(id='intermediate-value', style={'display': 'none'})
])

@app.callback(Output('intermediate-value', 'children'), [Input('dropdown', 'value')])
def clean_data(value):
     # some expensive clean data step
     cleaned_df = your_expensive_clean_or_compute_step(value)

     # more generally, this line would be
     # json.dumps(cleaned_df)
     return cleaned_df.to_json(date_format='iso', orient='split')

@app.callback(Output('graph', 'figure'), [Input('intermediate-value', 'children')])
def update_graph(jsonified_cleaned_data):

    # more generally, this line would be
    # json.loads(jsonified_cleaned_data)
    dff = pd.read_json(jsonified_cleaned_data, orient='split')

    figure = create_figure(dff)
    return figure

@app.callback(Output('table', 'children'), [Input('intermediate-value', 'children')])
def update_table(jsonified_cleaned_data):
    dff = pd.read_json(jsonified_cleaned_data, orient='split')
    table = create_table(dff)
    return table

```

## 例2 - 前もって集合体(aggregation)を計算する

ネットワーク越しに計算されたデータを転送するのは、もしそのデータが大きければ大きなコストがかかります。いくつかのケースでは、このデータをJSONに変換することもまたコストがかかってしまいます。  

多くの場合、アプリケーションはすでに計算されて抽出されたデータの集合体(aggregation)の一部を表示するだけでしょう。これらのケースでは、前もってデータ処理を行うコールバック内でその集合体(aggregations)を計算することができるでしょうし、また、これらの集合体(aggregations)を残りのコールバックに送ることもできるでしょう。  

ここではどのようにして抽出されて凝集されたデータを複数のコールバックに送るかということの簡単な例を見てみましょう。

```python
@app.callback(
    Output('intermediate-value', 'children'),
    [Input('dropdown', 'value')])
def clean_data(value):
     # an expensive query step
     cleaned_df = your_expensive_clean_or_compute_step(value)

     # a few filter steps that compute the data
     # as it's needed in the future callbacks
     df_1 = cleaned_df[cleaned_df['fruit'] == 'apples']
     df_2 = cleaned_df[cleaned_df['fruit'] == 'oranges']
     df_3 = cleaned_df[cleaned_df['fruit'] == 'figs']

     datasets = {
         'df_1': df_1.to_json(orient='split', date_format='iso'),
         'df_2': df_2.to_json(orient='split', date_format='iso'),
         'df_3': df_3.to_json(orient='split', date_format='iso'),
     }

     return json.dumps(datasets)

@app.callback(
    Output('graph', 'figure'),
    [Input('intermediate-value', 'children')])
def update_graph_1(jsonified_cleaned_data):
    datasets = json.loads(jsonified_cleaned_data)
    dff = pd.read_json(datasets['df_1'], orient='split')
    figure = create_figure_1(dff)
    return figure

@app.callback(
    Output('graph', 'figure'),
    [Input('intermediate-value', 'children')])
def update_graph_2(jsonified_cleaned_data):
    datasets = json.loads(jsonified_cleaned_data)
    dff = pd.read_json(datasets['df_2'], orient='split')
    figure = create_figure_2(dff)
    return figure

@app.callback(
    Output('graph', 'figure'),
    [Input('intermediate-value', 'children')])
def update_graph_3(jsonified_cleaned_data):
    datasets = json.loads(jsonified_cleaned_data)
    dff = pd.read_json(datasets['df_3'], orient='split')
    figure = create_figure_3(dff)
    return figure

```


## 例3 - キャッシング(Caching)とシグナリング(Signaling)

この例では:

- "グローバル変数"を保存するためにFlask-Cacheを通してRedisを使っています。このデータはその出力がキャッシュされて入力引数によってうまく調整された(keyed)関数を通してアクセスされます。  

- 負荷の高い計算が終わったときに他のコールバックにシグナルを送るために隠されたdivを使った方法を用います。  

- Redisの代わりにこれをファイルシステムに保存することもできます。詳しくはこちらを見てください: https://flask-caching.readthedocs.io/en/latest/  

- この"シグナリング"は素晴らしいものです。なぜなら、一つのプロセスを使うことだけに高負荷な計算をさせることができるからです。この種のシグナリングなしでは、各々のコールバックが1つではなく4つのプロセスを専有して並行に高負荷な計算を行ってしまうことになってしまいます。  

- このアプローチはまた将来のセッションが前もって計算された値を使うという利点ももっています。これは、少数の入力をもつアプリケーションではうまくいくでしょう。

この例がどんな感じのものなのかということを示しましょう。注意すべき点がいくつかあります:  

- time.sleep(5)を使っって高負荷なプロセスをシミュレートしています。  

- アプリケーションがロードされるとき、4つのグラフをレンダリングするために5秒かかっています。  

- 最初の計算は1つのプロセスをブロックするだけです。  

- 一度計算が終わると、シグナルが送られ、4つのコールバックがグラフをレンダリングするために並行に実行されます。これらの各コールバックは"グローバルストア(global store)"からデータを読み出してきます： ここではRedisキャッシュです。  

- 複数のコールバックが並行に実行されるようにapp.run_server内でprocesses=6と設定しました。製品内では、これは$ gunicorn --workers 6 --threads 2 app:server のような形で設定されることになります。  

- もし過去に選ばれたことがあれば、dropdownの値を選ぶことには5秒もかからないでしょう。これは、値がキャッシュからとってこられるからです。  

- 同様に、ページをロードしたり新しいウィンドウでアプリケーションを開くことも素早くおこなうことができます。なぜなら、初期状態や初期の高負荷な計算はすでに計算されてしまっているからです。

![](./plot1_chap5.gif) 

以下にコードを示します。

```python
import copy
import dash
from dash.dependencies import Input, Output
import dash_html_components as html
import dash_core_components as dcc
import datetime
from flask_caching import Cache
import numpy as np
import os
import pandas as pd
import time


app = dash.Dash(__name__)
CACHE_CONFIG = {
    # try 'filesystem' if you don't want to setup redis
    'CACHE_TYPE': 'redis',
    'CACHE_REDIS_URL': os.environ.get('REDIS_URL', 'localhost:6379')
}
cache = Cache()
cache.init_app(app.server, config=CACHE_CONFIG)

N = 100

df = pd.DataFrame({
    'category': (
        (['apples'] * 5 * N) +
        (['oranges'] * 10 * N) +
        (['figs'] * 20 * N) +
        (['pineapples'] * 15 * N)
    )
})
df['x'] = np.random.randn(len(df['category']))
df['y'] = np.random.randn(len(df['category']))

app.layout = html.Div([
    dcc.Dropdown(
        id='dropdown',
        options=[{'label': i, 'value': i} for i in df['category'].unique()],
        value='apples'
    ),
    html.Div([
        html.Div(dcc.Graph(id='graph-1'), className="six columns"),
        html.Div(dcc.Graph(id='graph-2'), className="six columns"),
    ], className="row"),
    html.Div([
        html.Div(dcc.Graph(id='graph-3'), className="six columns"),
        html.Div(dcc.Graph(id='graph-4'), className="six columns"),
    ], className="row"),

    # hidden signal value
    html.Div(id='signal', style={'display': 'none'})
])


# perform expensive computations in this "global store"
# these computations are cached in a globally available
# redis memory store which is available across processes
# and for all time.
@cache.memoize()
def global_store(value):
    # simulate expensive query
    print('Computing value with {}'.format(value))
    time.sleep(5)
    return df[df['category'] == value]


def generate_figure(value, figure):
    fig = copy.deepcopy(figure)
    filtered_dataframe = global_store(value)
    fig['data'][0]['x'] = filtered_dataframe['x']
    fig['data'][0]['y'] = filtered_dataframe['y']
    fig['layout'] = {'margin': {'l': 20, 'r': 10, 'b': 20, 't': 10}}
    return fig


@app.callback(Output('signal', 'children'), [Input('dropdown', 'value')])
def compute_value(value):
    # compute value and send a signal when done
    global_store(value)
    return value


@app.callback(Output('graph-1', 'figure'), [Input('signal', 'children')])
def update_graph_1(value):
    # generate_figure gets data from `global_store`.
    # the data in `global_store` has already been computed
    # by the `compute_value` callback and the result is stored
    # in the global redis cached
    return generate_figure(value, {
        'data': [{
            'type': 'scatter',
            'mode': 'markers',
            'marker': {
                'opacity': 0.5,
                'size': 14,
                'line': {'border': 'thin darkgrey solid'}
            }
        }]
    })


@app.callback(Output('graph-2', 'figure'), [Input('signal', 'children')])
def update_graph_2(value):
    return generate_figure(value, {
        'data': [{
            'type': 'scatter',
            'mode': 'lines',
            'line': {'shape': 'spline', 'width': 0.5},
        }]
    })


@app.callback(Output('graph-3', 'figure'), [Input('signal', 'children')])
def update_graph_3(value):
    return generate_figure(value, {
        'data': [{
            'type': 'histogram2d',
        }]
    })


@app.callback(Output('graph-4', 'figure'), [Input('signal', 'children')])
def update_graph_4(value):
    return generate_figure(value, {
        'data': [{
            'type': 'histogram2dcontour',
        }]
    })


# Dash CSS
app.css.append_css({
    "external_url": "https://codepen.io/chriddyp/pen/bWLwgP.css"})
# Loading screen CSS
app.css.append_css({
    "external_url": "https://codepen.io/chriddyp/pen/brPBPO.css"})

if __name__ == '__main__':
    app.run_server(debug=True, processes=6)
```
