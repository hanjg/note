## jsp中的js ##
- jsp里引入js一定要使用 **</script>** 补全标签，第二个写法无法加载js。
```jsp
<script src="/resources/js/jquery-3.3.1.js" type="text/javascript"></script>
<script src="/resources/js/jquery-3.3.1.js" type="text/javascript"/>
```
- jsp中用  <%=%> 传给js一定要加**单引号**，否则无法调用js。
```jsp
<a class="btn03" href="javascript:deleteBatch('<%=basePath%>')">
```
- js中var content = "",null,undefined,0；则if(content)为false。

## ajax requestbody ##
- 传list的格式，都好分割
```js
{parm:1,2,3}
```

## 精度 ##
- js的number超过**2的53次方**(十进制15位)，精确度就会失真