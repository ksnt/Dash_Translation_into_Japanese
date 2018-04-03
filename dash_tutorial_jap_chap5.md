# コールバック(Callbacks)間で状態を共有する

Dashのコアとなっている原理のうちの一つをここの入門ガイド（Getting Started Guide）のなかで説明します。それは、**Dash Callbacksは決してそのスコープ範囲外の変数から変更されてはいけない**ということです。この章ではなぜそれがいけないのかということを説明し、コールバック間での状態の共有のためのいくつかの代替的なパターンを提示します。

## なぜ状態を共有するのか？

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

複数のpythonプロセスに亘ってデータを安全に共有するためには各々のプロセスにアクセス可能などこかにデータをたくわえておく必要があります。このデータを格納するための場所は3つあります：

1 - ユーザーのブラウザのセッションの内

2 - ディスク上 (例えば、ファイルや新しいデータベース上)

3 - 共有メモリ空間（例えば、Redisと一緒に使う）

次の3つの例はこれらのアプローチを示す例となっています。


## 例1 - 隠されたDiv要素内にデータを格納する

To save data in user's browser's session:  

Implemented by saving the data as part of Dash's front-end store through methods explained in https://community.plot.ly/t/sharing-a-dataframe-between-plots/6173  
- Data has to be converted to a string like JSON for storage and transport  
- Data that is cached in this way will only be available in the user's current session.  
        If you open up a new browser, the app's callbacks will always compute the data. The data is only cached and transported between callbacks within the session.  
        As such, unlike with caching, this method doesn't increase the memory footprint of the app.  
        There could be a cost in network transport. If your sharing 10MB of data between callbacks, then that data will be transported over the network between each callback.  
        If the network cost is too high, then compute the aggregations upfront and transport those. Your app likely won't be displaying 10MB of data, it will just be displaying a subset or an aggregation of it.  

This example outlines how you can perform an expensive data processing step in one callback, serialize the output at JSON, and provide it as an input to the other callbacks. This example uses standard dash callbacks and stores the JSON-ified data inside a hidden div in the app.  
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

## 例2 - Computing Aggregations Upfront

Sending the computed data over the network can be expensive if the data is large. In some cases, serializing this data and JSON can also be expensive.  

In many cases, your app will only display a subset or an aggregation of the computed or filtered data. In these cases, you could precompute your aggregations in your data processing callback and transport these aggregations to the remaining callbacks.  

Here's a simple example of how you might transport filtered or aggregated data to multiple callbacks. 
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

This example:

- Uses Redis via Flask-Cache for storing “global variables”. This data is accessed through a function who’s output is cached and keyed by its input arguments.

- Uses the hidden div solution to send a signal to the other callbacks when the expensive computation is complete  

- Note that instead of Redis, you could also save this to the file system. See https://flask-caching.readthedocs.io/en/latest/ for more details.  

- This “signaling” is cool because it allows the expensive computation to only take up one process. Without this type of signaling, each callback could end up computing the expensive computation in parallel, locking 4 processes instead of 1.

This approach also has the advantage that future sessions use the pre-computed value. This will work well for apps that have a small number of inputs.

Here’s what this example looks like. Some things to note:  

- I’ve simulated an expensive process by using a time.sleep(5).  

- When the app loads, it takes 5 seconds to render all 4 graphs  

- The initial computation only blocks 1 process  

- Once the computation is complete, the signal is sent and 4 callbacks are executed in parallel to render the graphs. Each of these callbacks retrieves the data from the “global store”: the redis cache.  

- I’ve set processes=6 in app.run_server so that multiple callbacks can be executed in parallel. In production, this is done with something like $ gunicorn --workers 6 --threads 2 app:server  

- Selecting a value in the dropdown will take less than 5 seconds if it has already been selected in the past. This is because the value is being pulled from the cache.  

- Similarly, reloading the page or opening the app in a new window is also fast because the initial state and the initial expensive computation has already been computed.  

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
