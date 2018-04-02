# コールバック(Callbacks)間で状態(State)を共有する

Dashのコアとなっている原理のうちの一つをここの入門ガイド（Getting Started Guide）のなかで説明します。それは、**Dash Callbacksは決してそのスコープ範囲外の変数から変更されてはいけない**ということです。この章ではなぜそれがいけないのかということを説明し、コールバック間での状態の共有のためのいくつかの代替的なパターンを提示します。

## なぜ状態(State)を共有するのか？

いくつかのアプリケーションでは複数のコールバックをもつことができるかもしれません。そうしたコールバックはSQLクエリを発行したりシミュレーションを走らせたりデータをダウンロードするといったような高い負荷のかかるタスクに依存したものでしょう。

各々のコールバックに同じ高負荷なタスクを走らせる代わりに、一つのコールバックにタスクを走らせてから残りのコールバックにその結果を共有するさせることができます。


## なぜグローバル変数があなたのアプリケーションを壊すのか？


Dashはマルチユーザー環境で動くようにデザインされています。マルチユーザー環境というのは複数の人々が同時にアプリケーションを見たり独立のセッションを持ったりするような環境のことです。

もしあなたのアプリケーションが変更されたグローバル変数を使っていたら、ある一人のユーザーのセッションがその変数を別のユーザーに影響を与えてしまう変数に設定してしまうことができてしまいます。

Dashはまた複数のpython workersとともに走らせることが出来るようにデザインされています。そのことによってコールバックを並行に実行することができます。これは通常gnicornのシンタックスを用いて次のように行うことができます。

$ gunicorn --workers 4 --threads 2 app:server

Dashアプリケーションが複数のworkersにわたって動くとき、メモリが共有されることはありません。これは、もし一つのコールバックないのグローバル変数を修正したらその修正は残りのworkerに適用されることはないということです。

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

この例を直すためには、単純にフィルターをコールバック内の新しい変数に再度指定する、あるいはこのガイドの次の部分で述べられることになるストラテジーのうちの一つに従ってください。


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

複数のpythonプロセスに亘ってデータを安全に共有するためには各々のプロセスにアクセス可能などこかにデータをたくわえておく必要があります。このデータをたくわえておくための場所は3つあります：

1 - ユーザーのブラウザのセッションの内

2 - ディスク上 (例えば、ファイルや新しいデータベース上)

3 - 共有メモリ空間（例えば、Redisと一緒に使う）

次の3つの例はこれらのアプローチを示す例となっています。


## 例1 - 隠されたDiv要素ないにデータを貯める

**(TBA)**

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

## 例2

**(TBA)**

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


## 例3 - キャッシングとシグナリング(Caching and Signaling)

**(TBA)**


![](./plot1_chap5) 

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
