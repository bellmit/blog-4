
[廖雪峰：编写REST API](https://www.liaoxuefeng.com/wiki/1022910821149312/1105003357927328)


REST API规范
编写REST API，实际上就是编写处理HTTP请求的async函数，不过，REST请求和普通的HTTP请求有几个特殊的地方：

REST请求仍然是标准的HTTP请求，但是，除了GET请求外，POST、PUT等请求的body是JSON数据格式，请求的Content-Type为application/json；
REST响应返回的结果是JSON数据格式，因此，响应的Content-Type也是application/json。