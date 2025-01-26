# Web

`http.HandleFunc()`函数用于设置路由，`http.ListenAndServe()`函数用于启动Web服务。

`r.Form`字段保存了所有请求的参数，包括：

- URL 查询字符串（GET 请求）：例如 http://example.com/?name=John&age=30
- 表单数据（POST 请求）：例如通过 HTML 表单提交的数据

它的类型是 map[string][]string，是一个 url.Values 类型。在使用`r.Form`之前，必须先调用`r.ParseForm()`函数。

要增加上传文件功能，需要三步：

1. 在表单中增加 enctype="multipart/form-data" 属性
2. 使用`r.ParseMultipartForm()`函数解析请求体
3. 使用`r.FormFile("file")`获取上传的文件








