# 第五章 视图层实现

# 5 视图层实现

## 5.1 目录结构

在项目的 web 目录下新创建一个名为 `static` 的目录，用来存放 Web 应用视图层的所有静态内容

```go
$ cd $GOPATH/src/github.com/kongyixueyuan.com/education
$ mkdir web/static 
```

**`web/static`目录下包括四个子目录，分别为：**

*   `web/static/css` ：用于存放控制页面布局及显示样式所需的 `CSS` 文件
*   `web/static/js` ：用于存放编写的与用户交互的 `JavaScript` 源代码文件
*   `web/static/images`：用户存放页面显示所需的所有图片文件
*   `web/static/photo`：用于存储添加信息时上传的图片文件

```go
$ mkdir -p web/static/css
$ mkdir -p web/static/images
$ mkdir -p web/static/js
$ mkdir -p web/static/photo 
```

在项目的 web 目录下新创建一个名为 `tpl` 的目录，用来存放 Web 应用响应客户端的模板页面

```go
$ mkdir web/tpl 
```

在 `web/tpl` 目录下主要有如下页面：

*   `login.html`：用户登录页面
*   `index.html`：用户登录成功之后进入的首页面
*   `help.html`： 显示帮助信息及相关操作的链接页面
*   `query.html`：根据证书编号与姓名查询的页面
*   `query2.html`：根据身份证号码查询的页面
*   `queryResult.html`：根据不同的查询请求显示查询结果的页面
*   `addEdu.html`：添加信息的页面
*   `modify.html`：修改信息的页面

## 5.2 相关源码实现

相关源代码请参考：

CSS 部分：

[web/static/css/addEdu.css](https://github.com/kevin-hf/education/blob/master/web/static/css/addEdu.css)

[web/static/css/bootstrap.min.css](https://github.com/kevin-hf/education/blob/master/web/static/css/bootstrap.min.css)

[web/static/css/help.css](https://github.com/kevin-hf/education/blob/master/web/static/css/help.css)

[web/static/css/index.css](https://github.com/kevin-hf/education/blob/master/web/static/css/index.css)

[web/static/css/login.css](https://github.com/kevin-hf/education/blob/master/web/static/css/login.css)

[web/static/css/query.css](https://github.com/kevin-hf/education/blob/master/web/static/css/query.css)

[web/static/css/queryResult.css](https://github.com/kevin-hf/education/blob/master/web/static/css/queryResult.css)

[web/static/css/reset.css](https://github.com/kevin-hf/education/blob/master/web/static/css/reset.css)

JavaScript 部分

[web/static/js/bootstrap.min.js](https://github.com/kevin-hf/education/blob/master/web/static/js/bootstrap.min.js)

[web/static/js/jquery.min.js](https://github.com/kevin-hf/education/blob/master/web/static/js/jquery.min.js)

HTML 页面模板部分：

[web/tpl/addEdu.html](https://github.com/kevin-hf/education/blob/master/web/tpl/addEdu.html)

[web/tpl/help.html](https://github.com/kevin-hf/education/blob/master/web/tpl/help.html)

[web/tpl/index.html](https://github.com/kevin-hf/education/blob/master/web/tpl/index.html)

[web/tpl/login.html](https://github.com/kevin-hf/education/blob/master/web/tpl/login.html)

[web/tpl/modify.html](https://github.com/kevin-hf/education/blob/master/web/tpl/modify.html)

[web/tpl/query.html](https://github.com/kevin-hf/education/blob/master/web/tpl/query.html)

[web/tpl/query2.html](https://github.com/kevin-hf/education/blob/master/web/tpl/query2.html)

[web/tpl/queryResult.html](https://github.com/kevin-hf/education/blob/master/web/tpl/queryResult.html)

## 5.3 照片上传

在添加信息时需要额外实现一个功能－添加照片

使用 jQuery Ajax 功能实现

HTML 代码如下：

```go
 <div class="headImg">
                <div class="uploadImg">
                    <input type="file" name="" value="上传照片" id="file">
                    +
                    <!-- <img src="./images/head.jpg" alt=""> -->
                    <img src="" alt="">
                </div>
                <p>请上传照片(120*160px)</p>
            </div>
          </div> 
```

JavaScript 代码如下:

```go
 // 上传图片
        $('#file').unbind('change').bind('change',function() {
            event.stopPropagation();
            uploadFile('img');
            return;
        });
        // 头像图片
        var artImg;
        function uploadFile(type) {
            event.stopPropagation();
            let formData = new FormData();
            if( type == "img"){
                formData.append('file', $('#file')[0].files[0]);
            }
            $.ajax({
                url: '/upload',
                type: 'POST',
                cache: false,
                data: formData,
                processData: false,
                dataType: "json",
                contentType: false
            }).done(function (res) {
                if (res.error == "0") {
                    if( type == "img"){
                        $('.uploadImg img').attr('src',res.result.path);
                        $('#photo').val(res.result.path)
                        return artImg = res.result.path;
                    }
                } else {
                    alert("上传失败！" + res.result.msg)
                }
            }).fail(function (res) { });
        } 
```

在 `web/controller` 目录下创建一个 `upload.go` 文件

```go
$ vim web/controller/upload.go 
```

`upload.go` 文件主要利用 Ajax 完成 照片传功能的，完整代码如下：

```go
/**
  @Author : hanxiaodong
*/

package controller

import (
    "fmt"
    "net/http"
    "io/ioutil"
    "crypto/rand"
    "path/filepath"
    "os"
    "mime"
    "log"
)

func (app *Application) UploadFile(w http.ResponseWriter, r *http.Request)  {

    start := "{"
    content := ""
    end := "}"

    file, _, err := r.FormFile("file")
    if err != nil {
        content = "\"error\":1,\"result\":{\"msg\":\"指定了无效的文件\",\"path\":\"\"}"
        w.Write([]byte(start + content + end))
        return
    }
    defer file.Close()

    fileBytes, err := ioutil.ReadAll(file)
    if err != nil {
        content = "\"error\":1,\"result\":{\"msg\":\"无法读取文件内容\",\"path\":\"\"}"
        w.Write([]byte(start + content + end))
        return
    }

    filetype := http.DetectContentType(fileBytes)
    //log.Println("filetype = " + filetype)
    switch filetype {
    case "image/jpeg", "image/jpg":
    case "image/gif", "image/png":
    case "application/pdf":
        break
    default:
        content = "\"error\":1,\"result\":{\"msg\":\"文件类型错误\",\"path\":\"\"}"
        w.Write([]byte(start + content + end))
        return
    }

    fileName := randToken(12)    // 指定文件名
    fileEndings, err := mime.ExtensionsByType(filetype)    // 获取文件扩展名
    //log.Println("fileEndings = " + fileEndings[0])
    // 指定文件存储路径
    newPath := filepath.Join("web", "static", "photo", fileName + fileEndings[0])
    //fmt.Printf("FileType: %s, File: %s\n", filetype, newPath)

    newFile, err := os.Create(newPath)
    if err != nil {
        log.Println("创建文件失败：" + err.Error())
        content = "\"error\":1,\"result\":{\"msg\":\"创建文件失败\",\"path\":\"\"}"
        w.Write([]byte(start + content + end))
        return
    }
    defer newFile.Close()

    if _, err := newFile.Write(fileBytes); err != nil || newFile.Close() != nil {
        log.Println("写入文件失败：" + err.Error())
        content = "\"error\":1,\"result\":{\"msg\":\"保存文件内容失败\",\"path\":\"\"}"
        w.Write([]byte(start + content + end))
        return
    }

    path := "/static/photo/" + fileName + fileEndings[0]
    content = "\"error\":0,\"result\":{\"fileType\":\"image/png\",\"path\":\"" + path + "\",\"fileName\":\"ce73ac68d0d93de80d925b5a.png\"}"
    w.Write([]byte(start + content + end))
    return
}

func randToken(len int) string {
    b := make([]byte, len)
    rand.Read(b)
    return fmt.Sprintf("%x", b)
} 
```