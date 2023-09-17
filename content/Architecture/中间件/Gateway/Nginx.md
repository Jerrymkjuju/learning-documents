# Nginx 核心知识100讲笔记

## 初试Nginx

### 1. Nginx之root、alias区分

- 使用root，实际的路径就是：root值 + location值。

- 使用alias，实际的路径就是：alias值。

  > root的值会作为请求地址中的一部分。所以一般使用alias

另外：

- alias在使用正则匹配时，必须捕捉要匹配的内容，并在指定的内容处使用。

- alias只能位于location块中，root可以不放在location中

- alias的location参数不加/，那么path目录也可以不加，但是alias location加/，那么path目录必须加

