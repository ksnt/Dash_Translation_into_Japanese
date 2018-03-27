# Dash State

前章の[コールバックの基礎](https://github.com/ksnt/Dash_Translation_into_Japanese/blob/master/dash_tutorial_jap_chap2.md)ではコールバックはこんな感じでした：

```python
# -*- coding: utf-8 -*-
import dash
from dash.dependencies import Input, Output
import dash_core_components as dcc
import dash_html_components as html

app = dash.Dash(__name__)

app.layout = html.Div([
    dcc.Input(id='input-1', type='text', value='Montréal'),
    dcc.Input(id='input-2', type='text', value='Canada'),
    html.Div(id='output')
])


@app.callback(Output('output', 'children'),
              [Input('input-1', 'value'),
               Input('input-2', 'value')])
def update_output(input1, input2):
    return u'Input 1 is "{}" and Input 2 is "{}"'.format(input1, input2)


if __name__ == '__main__':
    app.run_server(debug=True)
```


(訳者注:上記のコードを実行して`localhost:8050`を訪れるとテキストボックスが表示されることを確認できるはずです)  


この例ではコールバック関数は`dash.dependencies.Input`によって指定された属性のどれか一つでも変化したらいつでも呼び出されます。  
テキストボックにデータを入れて確かめてみてください。  

`dash.dependencies.State`を使うことでコールバックを呼び起こすことなく追加の変数を与えることができます。次のコードは上のものと同じ例ですが、`dcc.Input`を`dash.dependencies.State`とともに使い、ボタンに対しては`dash.dependencies.Input`が用いられています。
```python
# -*- coding: utf-8 -*-
import dash
from dash.dependencies import Input, Output, State
import dash_core_components as dcc
import dash_html_components as html

app = dash.Dash()

app.layout = html.Div([
    dcc.Input(id='input-1-state', type='text', value='Montréal'),
    dcc.Input(id='input-2-state', type='text', value='Canada'),
    html.Button(id='submit-button', n_clicks=0, children='Submit'),
    html.Div(id='output-state')
])


@app.callback(Output('output-state', 'children'),
              [Input('submit-button', 'n_clicks')],
              [State('input-1-state', 'value'),
               State('input-2-state', 'value')])
def update_output(n_clicks, input1, input2):
    return u'''
        The Button has been pressed {} times,
        Input 1 is "{}",
        and Input 2 is "{}"
    '''.format(n_clicks, input1, input2)


if __name__ == '__main__':
    app.run_server(debug=True)
```

(訳者注:上記のコードを実行して`localhost:8050`を訪れるとテキストボックスとボタンが表示されることを確認できるはずです)   

この例では、`dcc.Input`ボックス内のテキストの変化はコールバックを呼び出しませんが、ボタンを押すとコールバックが呼び出されます。`dcc.Input`の現在の値は、たとえコールバック関数そのものを引き起こさなかったとしても、コールバックにわたされることになります。

今は`html.Button`コンポーネントの`n_clicks`属性を聴く（リッスンする）ことによりコールバックを呼び出しています。`n_clicks`はコンポーネントがクリックされるたびにインクリメントされる属性です。`dash_html_components`ライブラリ内の全てのコンポーネント内で利用可能です。
