# event-feed导出

## 利用 DOMjudge API 导出

- 打开浏览器，先在`Jury`界面确认好比赛的`CID`和`External ID`是什么，如果`External ID`不为空，则记住`External ID`对应的值；若为空，则记住`CID`对应的值。

  

- 原因在`https://github.com/DOMjudge/domjudge/wiki/Connecting-the-ICPC-Tools-with-DOMjudge`中有所解释（此时作者运行的是`docker`版的`domserver`和`judgehost`）:

  ```
  Note: if you run DOMjudge with the configuration option data_source set to either 1 or 2, you need to use the external ID of the contest instead of the CID. By default DOMjudge runs with data_source=0 and unless you know what you are doing you should not have to change this. However, DOMjudge Demoweb runs with data_source=1 so if you use that for testing use the external ID!
  ```

  

- 接下来，修改`http://[domjudge]/api/v4/contests/[ID]/event-feed?stream=false`，其中`[domjudge]`对应的是`DOMjudge`的首页网址，`[ID]`对应的是上一步中需要你记住的东西。
- 修改完成后，即可将该链接复制到浏览器地址栏中，然后键入回车，即可获得对应`ID`的`event-feed`了。

