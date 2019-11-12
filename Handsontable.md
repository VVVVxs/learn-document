# HandsonTable

## 一、安装

* 项目中直接引入handsontablen的依赖

``` shell
yarn add handsontable，
```

* 在react项目中需要引入

``` shell
yarn add @handsontable/react
```

* 在根组件需要引入handsontable的样式文件

```javascript
import 'handsontable/dist/handsontable.full.css';
```

## 二、开始上手 

* 在react项目中使用

```javascript
import React from 'react';
import { HotTable } from '@handsontable/react';

export default class App extends React.Component {
  constructor(props) {
    super(props);
    this.data = [
      ["", "Ford", "Volvo", "Toyota", "Honda"],
      ["2016", 10, 11, 12, 13],
      ["2017", 20, 11, 14, 13],
      ["2018", 30, 15, 12, 13]
    ];
  }

  render() {
    return (
      <div id="hot-app">
        <HotTable
            data={this.data} // 数据源
            colHeaders={true} // 是否显示列标题
            rowHeaders={true} // 是否显示行标题
            width="600" // 宽度 超出宽度默认出滚动条
            height="300" // 高度 超出高度默认出滚动条
        />
      </div>
    );
  }
}

```

* 不使用@handsonTable/react 提供的组件

``` javascript
import * as React from 'react';
import HandsonTable from 'handsontable';

export default class HandsonTableTest extends React.Component {
    constructor(props) {
        super(props);
        this.handTable = null;
    }
    componentDidMount() {
        if (this.handTable) {
            new HandsonTable(this.handTable, {
                data: HandsonTable.helper.createSpreadsheetData(5, 5),
                rowHeaders: true,
                colHeaders: true
            });
        }

    }
    render() {
        return (
            <div ref={(div) => this.handTable = div}></div>
        )
    }
}
```

<font color=red>*注意：</font>如果安装的是7.0版本以上的，会在表格下方提示一段购买商业许可证，所以还需要在加上licenseKey='non-commercial-and-evaluation',就不会再出现提示了。

```javascript
...
<HotTable
    data={this.data}
    colHeaders={true}
    rowHeaders={true}
    width="600"
    height="300"
    licenseKey='non-commercial-and-evaluation'
/>
```
