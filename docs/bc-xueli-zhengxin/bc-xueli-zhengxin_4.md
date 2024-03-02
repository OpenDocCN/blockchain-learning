# 第四章 控制层实现

# 4 控制层实现

## 4.1 设置系统用户

通过业务层已经实现了利用 `fabric-sdk-go` 调用链码查询或操作分类账本状态，接下来，我们开始实现 Web 应用层，应用层将其分为两个部分，

*   **控制层**
*   **视图层**

在项目根目录下新创建一个名为 `web` 的目录，用来存放 Web 应用层的所有内容

```go
$ cd $GOPATH/src/github.com/kongyixueyuan.com/education
$ mkdir -p web/controller 
```

在 `web` 目录下创建 `controller` 子目录，在该目录下创建 `userInfo.go` 、 `controllerResponse.go` 与 `controllerHandler.go` 三个文件

```go
$ vim web/controller/userInfo.go 
```

`userInfo.go` 用来模拟 RDB，保存系统用户信息，作为用户登录时核对用户信息，当然，这部分大家可以使用 `MySQL` 或其它数据库来实现。

`userInfo.go` 完整代码如下：

```go
/**
  @Author : hanxiaodong
*/

package controller

import "github.com/kongyixueyuan.com/education/service"

type Application struct {
    Setup *service.ServiceSetup
}

type User struct {
    LoginName    string
    Password    string
    IsAdmin    string
}

var users []User

func init() {

    admin := User{LoginName:"Hanxiaodong", Password:"123456", IsAdmin:"T"}
    alice := User{LoginName:"ChainDesk", Password:"123456", IsAdmin:"T"}
    bob := User{LoginName:"alice", Password:"123456", IsAdmin:"F"}
    jack := User{LoginName:"bob", Password:"123456", IsAdmin:"F"}

    users = append(users, admin)
    users = append(users, alice)
    users = append(users, bob)
    users = append(users, jack)

}

func isAdmin(cuser User) bool {
    if cuser.IsAdmin == "T"{
        return true
    }
    return false
} 
```

## 4.2 处理响应

创建 `controllerResponse.go` 文件

```go
$ vim web/controller/controllerResponse.go 
```

`controllerResponse.go` 主要实现对用户请求的响应，将响应结果返回给客户端浏览器。文件完整代码如下：

```go
/**
  @Author : hanxiaodong
*/

package controller

import (
    "net/http"
    "path/filepath"
    "html/template"
    "fmt"
)

func ShowView(w http.ResponseWriter, r *http.Request, templateName string, data interface{})  {

    // 指定视图所在路径
    pagePath := filepath.Join("web", "tpl", templateName)

    resultTemplate, err := template.ParseFiles(pagePath)
    if err != nil {
        fmt.Printf("创建模板实例错误: %v", err)
        return
    }

    err = resultTemplate.Execute(w, data)
    if err != nil {
        fmt.Printf("在模板中融合数据时发生错误: %v", err)
        //fmt.Fprintf(w, "显示在客户端浏览器中的错误信息")
        return
    }

} 
```

## 4.3 处理请求

创建 `controllerHandler.go` 文件

```go
$ vim web/controller/controllerHandler.go 
```

`controllerHandler.go` 文件主要实现接收用户请求，并根据不同的用户请求调用业务层不同的函数，实现对分类账本的访问。其中需要声明并实现的函数：

文件完整内容如下：

```go
/**
  @Author : hanxiaodong
*/

package controller

import (
    "net/http"
    "encoding/json"
    "github.com/kongyixueyuan.com/education/service"
    "fmt"
)

var cuser User

func (app *Application) LoginView(w http.ResponseWriter, r *http.Request)  {

    ShowView(w, r, "login.html", nil)
}

func (app *Application) Index(w http.ResponseWriter, r *http.Request)  {
    ShowView(w, r, "index.html", nil)
}

func (app *Application) Help(w http.ResponseWriter, r *http.Request)  {
    data := &struct {
        CurrentUser User
    }{
        CurrentUser:cuser,
    }
    ShowView(w, r, "help.html", data)
}

// 用户登录
func (app *Application) Login(w http.ResponseWriter, r *http.Request) {
    loginName := r.FormValue("loginName")
    password := r.FormValue("password")

    var flag bool
    for _, user := range users {
        if user.LoginName == loginName && user.Password == password {
            cuser = user
            flag = true
            break
        }
    }

    data := &struct {
        CurrentUser User
        Flag bool
    }{
        CurrentUser:cuser,
        Flag:false,
    }

    if flag {
        // 登录成功
        ShowView(w, r, "index.html", data)
    }else{
        // 登录失败
        data.Flag = true
        data.CurrentUser.LoginName = loginName
        ShowView(w, r, "login.html", data)
    }
}

// 用户登出
func (app *Application) LoginOut(w http.ResponseWriter, r *http.Request)  {
    cuser = User{}
    ShowView(w, r, "login.html", nil)
}

// 显示添加信息页面
func (app *Application) AddEduShow(w http.ResponseWriter, r *http.Request)  {
    data := &struct {
        CurrentUser User
        Msg string
        Flag bool
    }{
        CurrentUser:cuser,
        Msg:"",
        Flag:false,
    }
    ShowView(w, r, "addEdu.html", data)
}

// 添加信息
func (app *Application) AddEdu(w http.ResponseWriter, r *http.Request)  {

    edu := service.Education{
        Name:r.FormValue("name"),
        Gender:r.FormValue("gender"),
        Nation:r.FormValue("nation"),
        EntityID:r.FormValue("entityID"),
        Place:r.FormValue("place"),
        BirthDay:r.FormValue("birthDay"),
        EnrollDate:r.FormValue("enrollDate"),
        GraduationDate:r.FormValue("graduationDate"),
        SchoolName:r.FormValue("schoolName"),
        Major:r.FormValue("major"),
        QuaType:r.FormValue("quaType"),
        Length:r.FormValue("length"),
        Mode:r.FormValue("mode"),
        Level:r.FormValue("level"),
        Graduation:r.FormValue("graduation"),
        CertNo:r.FormValue("certNo"),
        Photo:r.FormValue("photo"),
    }

    app.Setup.SaveEdu(edu)

    r.Form.Set("certNo", edu.CertNo)
    r.Form.Set("name", edu.Name)
    app.FindCertByNoAndName(w, r)
}

func (app *Application) QueryPage(w http.ResponseWriter, r *http.Request)  {
    data := &struct {
        CurrentUser User
        Msg string
        Flag bool
    }{
        CurrentUser:cuser,
        Msg:"",
        Flag:false,
    }
    ShowView(w, r, "query.html", data)
}

// 根据证书编号与姓名查询信息
func (app *Application) FindCertByNoAndName(w http.ResponseWriter, r *http.Request)  {
    certNo := r.FormValue("certNo")
    name := r.FormValue("name")
    result, err := app.Setup.FindEduByCertNoAndName(certNo, name)
    var edu = service.Education{}
    json.Unmarshal(result, &edu)

    fmt.Println("根据证书编号与姓名查询信息成功：")
    fmt.Println(edu)

    data := &struct {
        Edu service.Education
        CurrentUser User
        Msg string
        Flag bool
        History bool
    }{
        Edu:edu,
        CurrentUser:cuser,
        Msg:"",
        Flag:false,
        History:false,
    }

    if err != nil {
        data.Msg = err.Error()
        data.Flag = true
    }

    ShowView(w, r, "queryResult.html", data)
}

func (app *Application) QueryPage2(w http.ResponseWriter, r *http.Request)  {
    data := &struct {
        CurrentUser User
        Msg string
        Flag bool
    }{
        CurrentUser:cuser,
        Msg:"",
        Flag:false,
    }
    ShowView(w, r, "query2.html", data)
}

// 根据身份证号码查询信息
func (app *Application) FindByID(w http.ResponseWriter, r *http.Request)  {
    entityID := r.FormValue("entityID")
    result, err := app.Setup.FindEduInfoByEntityID(entityID)
    var edu = service.Education{}
    json.Unmarshal(result, &edu)

    data := &struct {
        Edu service.Education
        CurrentUser User
        Msg string
        Flag bool
        History bool
    }{
        Edu:edu,
        CurrentUser:cuser,
        Msg:"",
        Flag:false,
        History:true,
    }

    if err != nil {
        data.Msg = err.Error()
        data.Flag = true
    }

    ShowView(w, r, "queryResult.html", data)
}

// 修改/添加新信息
func (app *Application) ModifyShow(w http.ResponseWriter, r *http.Request)  {
    // 根据证书编号与姓名查询信息
    certNo := r.FormValue("certNo")
    name := r.FormValue("name")
    result, err := app.Setup.FindEduByCertNoAndName(certNo, name)

    var edu = service.Education{}
    json.Unmarshal(result, &edu)

    data := &struct {
        Edu service.Education
        CurrentUser User
        Msg string
        Flag bool
    }{
        Edu:edu,
        CurrentUser:cuser,
        Flag:true,
        Msg:"",
    }

    if err != nil {
        data.Msg = err.Error()
        data.Flag = true
    }

    ShowView(w, r, "modify.html", data)
}

// 修改/添加新信息
func (app *Application) Modify(w http.ResponseWriter, r *http.Request) {
    edu := service.Education{
        Name:r.FormValue("name"),
        Gender:r.FormValue("gender"),
        Nation:r.FormValue("nation"),
        EntityID:r.FormValue("entityID"),
        Place:r.FormValue("place"),
        BirthDay:r.FormValue("birthDay"),
        EnrollDate:r.FormValue("enrollDate"),
        GraduationDate:r.FormValue("graduationDate"),
        SchoolName:r.FormValue("schoolName"),
        Major:r.FormValue("major"),
        QuaType:r.FormValue("quaType"),
        Length:r.FormValue("length"),
        Mode:r.FormValue("mode"),
        Level:r.FormValue("level"),
        Graduation:r.FormValue("graduation"),
        CertNo:r.FormValue("certNo"),
        Photo:r.FormValue("photo"),
    }

    app.Setup.ModifyEdu(edu)

    r.Form.Set("entityID", edu.EntityID)
    app.FindByID(w, r)
} 
```

> 提示：用户在做一些管理操作时需要验证其它是否有相应的操作权限，需要另外进行设计。

## 4.4 指定路由

在 `web` 目录下创建一个 `webServer.go` 文件

```go
$ vim web/webServer.go 
```

该文件主要声明用户请求的路由信息，并且指定 Web 服务的启动信息。文件完整内容如下：

```go
/**
  @Author : hanxiaodong
*/

package web

import (
    "net/http"
    "fmt"
    "github.com/kongyixueyuan.com/education/web/controller"
)

// 启动 Web 服务并指定路由信息
func WebStart(app controller.Application)  {

    fs:= http.FileServer(http.Dir("web/static"))
    http.Handle("/static/", http.StripPrefix("/static/", fs))

    // 指定路由信息(匹配请求)
    http.HandleFunc("/", app.LoginView)
    http.HandleFunc("/login", app.Login)
    http.HandleFunc("/loginout", app.LoginOut)

    http.HandleFunc("/index", app.Index)
    http.HandleFunc("/help", app.Help)

    http.HandleFunc("/addEduInfo", app.AddEduShow)    // 显示添加信息页面
    http.HandleFunc("/addEdu", app.AddEdu)    // 提交信息请求

    http.HandleFunc("/queryPage", app.QueryPage)    // 转至根据证书编号与姓名查询信息页面
    http.HandleFunc("/query", app.FindCertByNoAndName)    // 根据证书编号与姓名查询信息

    http.HandleFunc("/queryPage2", app.QueryPage2)    // 转至根据身份证号码查询信息页面
    http.HandleFunc("/query2", app.FindByID)    // 根据身份证号码查询信息

    http.HandleFunc("/modifyPage", app.ModifyShow)    // 修改信息页面
    http.HandleFunc("/modify", app.Modify)    //  修改信息

    http.HandleFunc("/upload", app.UploadFile)

    fmt.Println("启动 Web 服务, 监听端口号为: 9000")
    err := http.ListenAndServe(":9000", nil)
    if err != nil {
        fmt.Printf("Web 服务启动失败: %v", err)
    }

} 
```