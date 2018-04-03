## インストール

ターミナル上でいくつかのdashライブラリをインストールしてください。これらのライブラリは開発中のものなのでインストールとアップグレードを頻繁に行うことを推奨します。Python2とPython3をサポートしています。

> pip install dash==0.21.0  # The core dash backend  
> pip install dash-renderer==0.11.3  # The dash front-end 
> pip install dash-html-components==0.9.0  # HTML components  
> pip install dash-core-components==0.20.2  # Supercharged components  
> pip install plotly --upgrade  # Plotly graphing library used in examples  

## 対話性

### Dashアプリケーションレイアウト

```python
import dash
from dash.dependencies import Input, Output
import dash_core_components as dcc
import dash_html_components as html

app = dash.Dash()

app.layout = html.Div([
    dcc.Input(id='my-id', value='initial value', type='text'),
    html.Div(id='my-div')
])


@app.callback(
    Output(component_id='my-div', component_property='children'),
    [Input(component_id='my-id', component_property='value')]
)
def update_output_div(input_value):
    return 'You\'ve entered "{}"'.format(input_value)


if __name__ == '__main__':
    app.run_server()
```
![](./newplot1_chap2.png)

(訳者注:上記のコードを実行してlocalhost:8050を訪れるとテキストボックスが表示されることを確認できるはずです)

テキストボックスに何かタイプしてみてください。出力コンポーネントの子(children)はただちに更新されます。ここで起こっていることを分解してみましょう：

1.われわれのアプリケーションインターフェースの"入力"と"出力"は`app.callback`デコレーターを通じて宣言的に記述されます。  

2.Dashではアプリケーションの入力と出力は単純に特定のコンポーネントの属性となっています。この例では、入力は"`my-id`"というIDをもったコンポーネントの"value"属性です。出力は"`my-id`"というIDをともなうコンポーネントの"`children`"属性です。  

3.入力属性が変わると、かならずコールバックデコレータがラップ(wrap)する関数が自動的に呼ばれることになります。Dashは入力属性の新しい値をもった関数を入力引数として与えます。また、Dashは関数として帰ってきたあらゆるものとともに出力コンポーネントの属性を更新します。  

4.`component_id`および`component_property`キーワードはオプショナルです（それらのオブジェクト各々のために2つの引数が存在しているだけです）ここではそれらの属性を明快さのために入れることにしましたが簡潔さと読みやすさのために今後は省力することでしょう。

5.`dash.dependencies.Input`オブジェクトと`dash_core_components.Input`オブジェクトを混合しないでください。前者はコールバック内で使われるているだけで、後者は実際にコンポーネントなのです。

6.`layout`内の`my-div`コンポーネントの`children`属性のための値をどのようにセットしないようにしえいるかということに注目してください。Dashアプリケーションが起動するとき、出力コンポーネントの初期状態を格納する(populate)ために入力コンポーネントの初期値といっしょにコールバックの全てを自動的に呼び出します。

マイクロソフトのExcelを使ったプログミングのようなものです：入力セルが変わればいつでもそのセルに関係している全てのセルが自動的に更新されます。これは"リアクティブプログラミング(Reactive Programming)"と呼ばれています。

どのようにして全てのコンポーネントがキーワード引数を通じて記述されたかを覚えていますか？ 今、それらの属性が重要なのです。Dashの対話性をもってして、われわれはコールバック関数を通じてそれらの属性の全てを動的に更新することが出来るのです。しばしば我々は新しいテキストを表示するためにコンポーネントの`children`を更新するでしょう。あるいは、新しいデータを表示するために`dcc.Graph`コンポーネントの`figure`を更新することもあるでしょう。しかし、われわれはコンポーネントの`style`をアップデートするも、さらには`dcc.Dropdown`コンポーネントの`options`さえも更新することが出来るのです！

さあ、`dcc.Slider`が`dcc.Graph`を更新する別の例を見てみましょう。


```python
import dash
import dash_core_components as dcc
import dash_html_components as html
import plotly.graph_objs as go
import pandas as pd

df = pd.read_csv(
    'https://raw.githubusercontent.com/plotly/'
    'datasets/master/gapminderDataFiveYear.csv')

app = dash.Dash()

app.layout = html.Div([
    dcc.Graph(id='graph-with-slider'),
    dcc.Slider(
        id='year-slider',
        min=df['year'].min(),
        max=df['year'].max(),
        value=df['year'].min(),
        step=None,
        marks={str(year): str(year) for year in df['year'].unique()}
    )
])


@app.callback(
    dash.dependencies.Output('graph-with-slider', 'figure'),
    [dash.dependencies.Input('year-slider', 'value')])
def update_figure(selected_year):
    filtered_df = df[df.year == selected_year]
    traces = []
    for i in filtered_df.continent.unique():
        df_by_continent = filtered_df[filtered_df['continent'] == i]
        traces.append(go.Scatter(
            x=df_by_continent['gdpPercap'],
            y=df_by_continent['lifeExp'],
            text=df_by_continent['country'],
            mode='markers',
            opacity=0.7,
            marker={
                'size': 15,
                'line': {'width': 0.5, 'color': 'white'}
            },
            name=i
        ))

    return {
        'data': traces,
        'layout': go.Layout(
            xaxis={'type': 'log', 'title': 'GDP Per Capita'},
            yaxis={'title': 'Life Expectancy', 'range': [20, 90]},
            margin={'l': 40, 'b': 40, 't': 10, 'r': 10},
            legend={'x': 0, 'y': 1},
            hovermode='closest'
        )
    }


if __name__ == '__main__':
    app.run_server()
```
![](./newplot2_chap2.png)

(訳者注:上記のコードを実行してlocalhost:8050を訪れると上のような対話可能なグラフが表示されることを確認できるはずです)

この例では、`Slider`の`"value"`属性がアプリケーションの入力で`Graph`の`"figure"`属性がアプリケーションの出力になっています。`Slider`の`value`が変わるときはいつDashは新しい値とともに`update_figure`というコールバック関数を呼び出します。この関数は新しい値を持ったデータフレームを抽出し、`figure`オブジェクトをつくり、それをDashアプリケーションに返します。  

この例の中にはいくつかのすばらしいパターンが含まれています： 

1. We're using the Pandas library for importing and filtering datasets in memory.　　
2. We load our dataframe at the start of the app: df = pd.read_csv('...'). This dataframe df is in the global state of the app and can be read inside the callback functions.  
3. Loading data into memory can be expensive. By loading querying data at the start of the app instead of inside the callback functions, we ensure that this operation is only done when the app server starts. When a user visits the app or interacts with the app, that data (the df) is already in memory. If possible, expensive initialization (like downloading or querying data) should be done in the global scope of the app instead of within the callback functions.  
4.The callback does not modify the original data, it just creates copies of the dataframe by filtered through pandas filters. This is important: your callbacks should never mutate variables outside of their scope. If your callbacks modify global state, then one user's session might affect the next user's session and when the app is deployed on multiple processes or threads, those modifications will not be shared across sessions.  
    
### 複数入力

In Dash any "Output" can have multiple "Input" components. Here's a simple example that binds 5 Inputs (the value property of 2 Dropdown components, 2 RadioItems components, and 1 Slider component) to 1 Output component (the figure property of the Graph component). Notice how the app.callback lists all 5 dash.dependencies.Input inside a list in the second argument.


```python
import dash
import dash_core_components as dcc
import dash_html_components as html
import plotly.graph_objs as go
import pandas as pd

app = dash.Dash()

df = pd.read_csv(
    'https://gist.githubusercontent.com/chriddyp/'
    'cb5392c35661370d95f300086accea51/raw/'
    '8e0768211f6b747c0db42a9ce9a0937dafcbd8b2/'
    'indicators.csv')

available_indicators = df['Indicator Name'].unique()

app.layout = html.Div([
    html.Div([

        html.Div([
            dcc.Dropdown(
                id='xaxis-column',
                options=[{'label': i, 'value': i} for i in available_indicators],
                value='Fertility rate, total (births per woman)'
            ),
            dcc.RadioItems(
                id='xaxis-type',
                options=[{'label': i, 'value': i} for i in ['Linear', 'Log']],
                value='Linear',
                labelStyle={'display': 'inline-block'}
            )
        ],
        style={'width': '48%', 'display': 'inline-block'}),

        html.Div([
            dcc.Dropdown(
                id='yaxis-column',
                options=[{'label': i, 'value': i} for i in available_indicators],
                value='Life expectancy at birth, total (years)'
            ),
            dcc.RadioItems(
                id='yaxis-type',
                options=[{'label': i, 'value': i} for i in ['Linear', 'Log']],
                value='Linear',
                labelStyle={'display': 'inline-block'}
            )
        ],style={'width': '48%', 'float': 'right', 'display': 'inline-block'})
    ]),

    dcc.Graph(id='indicator-graphic'),

    dcc.Slider(
        id='year--slider',
        min=df['Year'].min(),
        max=df['Year'].max(),
        value=df['Year'].max(),
        step=None,
        marks={str(year): str(year) for year in df['Year'].unique()}
    )
])

@app.callback(
    dash.dependencies.Output('indicator-graphic', 'figure'),
    [dash.dependencies.Input('xaxis-column', 'value'),
     dash.dependencies.Input('yaxis-column', 'value'),
     dash.dependencies.Input('xaxis-type', 'value'),
     dash.dependencies.Input('yaxis-type', 'value'),
     dash.dependencies.Input('year--slider', 'value')])
def update_graph(xaxis_column_name, yaxis_column_name,
                 xaxis_type, yaxis_type,
                 year_value):
    dff = df[df['Year'] == year_value]

    return {
        'data': [go.Scatter(
            x=dff[dff['Indicator Name'] == xaxis_column_name]['Value'],
            y=dff[dff['Indicator Name'] == yaxis_column_name]['Value'],
            text=dff[dff['Indicator Name'] == yaxis_column_name]['Country Name'],
            mode='markers',
            marker={
                'size': 15,
                'opacity': 0.5,
                'line': {'width': 0.5, 'color': 'white'}
            }
        )],
        'layout': go.Layout(
            xaxis={
                'title': xaxis_column_name,
                'type': 'linear' if xaxis_type == 'Linear' else 'log'
            },
            yaxis={
                'title': yaxis_column_name,
                'type': 'linear' if yaxis_type == 'Linear' else 'log'
            },
            margin={'l': 40, 'b': 40, 't': 10, 'r': 0},
            hovermode='closest'
        )
    }


if __name__ == '__main__':
    app.run_server()
```

![](./newplot3_chap2.png)


In this example, the update_graph function gets called whenever the value property of the Dropdown, Slider, or RadioItems components change.  

The input arguments of the update_graph function are the new or current value of the each of the Input properties, in the order that they were specified.  

Even though only a single Input changes at a time (a user can only change the value of a single Dropdown in a given moment), Dash collects the current state of all of the specified Input properties and passes them into your function for you. Your callback functions are always guaranteed to be passed the representative state of the app.  

Let's extend our example to include multiple outputs.  

### 複数出力

Each Dash callback function can only update a single Output property. To update multiple Outputs, just write multiple functions.


```python
import dash
import dash_core_components as dcc
import dash_html_components as html

app = dash.Dash('')

app.layout = html.Div([
    dcc.RadioItems(
        id='dropdown-a',
        options=[{'label': i, 'value': i} for i in ['Canada', 'USA', 'Mexico']],
        value='Canada'
    ),
    html.Div(id='output-a'),

    dcc.RadioItems(
        id='dropdown-b',
        options=[{'label': i, 'value': i} for i in ['MTL', 'NYC', 'SF']],
        value='MTL'
    ),
    html.Div(id='output-b')

])


@app.callback(
    dash.dependencies.Output('output-a', 'children'),
    [dash.dependencies.Input('dropdown-a', 'value')])
def callback_a(dropdown_value):
    return 'You\'ve selected "{}"'.format(dropdown_value)


@app.callback(
    dash.dependencies.Output('output-b', 'children'),
    [dash.dependencies.Input('dropdown-b', 'value')])
def callback_b(dropdown_value):
    return 'You\'ve selected "{}"'.format(dropdown_value)


if __name__ == '__main__':
    app.run_server()
```

![](./newplot4_chap2.png)

You can also chain outputs and inputs together: the output of one callback function could be the input of another callback function.  

This pattern can be used to create dynamic UIs where one input component updates the available options of the next input component. Here's a simple example.  

```python
# -*- coding: utf-8 -*-
import dash
from dash.dependencies import Input, Output
import dash_core_components as dcc
import dash_html_components as html

app = dash.Dash(__name__)

all_options = {
    'America': ['New York City', 'San Francisco', 'Cincinnati'],
    'Canada': [u'Montréal', 'Toronto', 'Ottawa']
}
app.layout = html.Div([
    dcc.RadioItems(
        id='countries-dropdown',
        options=[{'label': k, 'value': k} for k in all_options.keys()],
        value='America'
    ),

    html.Hr(),

    dcc.RadioItems(id='cities-dropdown'),

    html.Hr(),

    html.Div(id='display-selected-values')
])


@app.callback(
    dash.dependencies.Output('cities-dropdown', 'options'),
    [dash.dependencies.Input('countries-dropdown', 'value')])
def set_cities_options(selected_country):
    return [{'label': i, 'value': i} for i in all_options[selected_country]]


@app.callback(
    dash.dependencies.Output('cities-dropdown', 'value'),
    [dash.dependencies.Input('cities-dropdown', 'options')])
def set_cities_value(available_options):
    return available_options[0]['value']


@app.callback(
    dash.dependencies.Output('display-selected-values', 'children'),
    [dash.dependencies.Input('countries-dropdown', 'value'),
     dash.dependencies.Input('cities-dropdown', 'value')])
def set_display_children(selected_country, selected_city):
    return u'{} is a city in {}'.format(
        selected_city, selected_country,
    )


if __name__ == '__main__':
    app.run_server(debug=True)
```
The first callback updates the available options in the second RadioItems component based off of the selected value in the first RadioItems component.  

The second callback sets an initial value when the options property changes: it sets it to the first value in that options array.  

The final callback displays the selected value of each component. If you change the value of the countries RadioItems component, Dash will wait until the value of the cities component is updated before calling the final callback. This prevents your callbacks from being called with inconsistent state like with "USA" and "Montréal".  

## まとめ

Dashにおけるコールバックの基礎についてカバーしました。Dashアプリケーションは単純ではあるもののパワフルな原理から作りあげられています。それらの原理とは、リアクティブかつファンクショナルなPythonコールバックを通じてカスタム可能な宣言的UIの数々です。宣言的なコンポーネントのすべての要素属性はコールバックを通じて更新することができます。また、`dcc.Dropdown`の`value`属性のような一部の属性はインターフェース内でユーザーが編集することができます。

------------------------------------------------------

次の章では、ページ上のグラフとの対話に対応するアプリケーションをつくるために、`dash_core_components.Graph`コンポーネントと一緒にこうした原理をどのように使うのかということを説明します。

[Part 3 - Interactive Graphing](dash_tutorial_jap_chap3.md)
