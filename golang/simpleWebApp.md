## 写一个简单 Web 应用
### 1. 介绍

**1.1 涉及到的知识要点：**

- 创建一个实现了save和load方法的结构体
- 使用 net/http 包来构建web服务
- 使用 html/template 包来处理HTML模板
- 使用 regexp 包来验证用户输入
- 使用了闭包

**1.2 需要具备的知识：**

- 有编程经验
- 理解web开发的基础知识 (HTTP, HTML)
- 了解一些Linux命令行

### 2. 开始

**2.1 数据结构**

通常一个博客文章页面包括文章标题和文章内容，因此定义一个有title和body属性的结构体。
```go
type Page struct {
    Title string
    Body  []byte
}
```
Page结构体描述了一个页面在内存中的存储格式，为了能够持久的保存页面数据，可以为Page实现一个save方法。
```go
func (p *Page) save() error {
    filename := p.Title + ".txt"
    return ioutil.WriteFile(filename, p.Body, 0600)
}
```
save方法将Page页面的内容保存在了一个文本文件，Page的标题是文件的文件名。save方法的第三个参数`0600`表示创建的文件的具有读写权限。

除了保存页面，还需要加载页面：
```go
func loadPage(title string) (*Page, error) {
    filename := title + ".txt"
    body, err := ioutil.ReadFile(filename)
    if err != nil {
        return nil, err
    }
    return &Page{Title: title, Body: body}, nil
}
```

现在我们有了简单的Page结构体，并能够将Page保存到文件和从文件中加载数据到Page中：
```go
func main() {
    p1 := &Page{Title: "TestPage", Body: []byte("This is a sample Page.")}
    p1.save()
    p2, _ := loadPage("TestPage")
    fmt.Println(string(p2.Body))
}
```

**2.2 介绍 net/http 包**

如下是一个简单web服务的实例：
```go
package main

import (
    "fmt"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
}

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
```
`http.HandleFunc("/", handler)`将url`"/"`和`handler`绑定。意味着http将使用handler函数来处理所有http请求。
`http.ListenAndServe(":8080", nil)`该函数调用后web服务就开始在8080端口启动监听请求。<br>
handler函数有两个参数，类型为http.ResponseWriter和*http.Request;http.ResponseWriter值代表web服务器的响应，通过将数据写入w，数据就会发送到客户端。http.Request 结构体代表客户端的请求，可以通过r来获取客户端发送给web服务的数据。r.Url.Path 代表请求的uri。

**2.3 使用net/http包写web服务**

首先引入 net/http 包：
```go
import (
	"fmt"
	"io/ioutil"
	"net/http"
)
```

创建一个 viewHandler, 用于客户端查看页面. 请求的uri前缀是 "/view/".
```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, _ := loadPage(title)
    fmt.Fprintf(w, "<h1>%s</h1><div>%s</div>", p.Title, p.Body)
}
```
使用该handler，我们重写main函数来初始化http使用viewHandler来处理所有以 “/view/” 开头的url请求。
```go
func main() {
    http.HandleFunc("/view/", viewHandler)
    http.ListenAndServe(":8080", nil)
}
```
接着在工程目录下新建一个文件，入test.txt , 并写入“my first web app”，然后编译代码并执行。此时我们可以访问 http://localhost:8080/view/test ，浏览器会显示 "my first web app"。

**2.4 页面编辑和保存**

上一步我们实现了页面的查看，接下来实现页面的编辑和保存。首先编写一个`editHandler`来实现页面的编辑:
```go
func editHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/edit/"):]
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    fmt.Fprintf(w, "<h1>Editing %s</h1>"+
        "<form action=\"/save/%s\" method=\"POST\">"+
        "<textarea name=\"body\">%s</textarea><br>"+
        "<input type=\"submit\" value=\"Save\">"+
        "</form>",
        p.Title, p.Title, p.Body)
}
```

页面编辑完成后需要将表单数据提交保存，接着编写`saveHandler`来保存表单数据：
```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/save/"):]
    body := r.FormValue("body")
    p := &Page{Title: title, Body: []byte(body)}
    p.save()
    http.Redirect(w, r, "/view/"+title, http.StatusFound)
}
```

在main函数中添加到`editHandler`和 `saveHandler`的路由：
```go
func main() {
    http.HandleFunc("/view/", viewHandler)
    http.HandleFunc("/edit/", editHandler)
    http.HandleFunc("/save/", saveHandler)
    http.ListenAndServe(":8080", nil)
}
```

到目前为止我们就已经完成了基本的web服务，该web服务能够实现页面的编辑，保存和查看。但是HTML数据是直接输出的，所以一旦html数据变得复杂时将无法维护代码。所以我们需要通过html模板来实现html文件和代码的分离。

**2.5 html/template包**

html/template 包是go语言标准库的一部分，我们可以利用 html/template 来实现HTML文件和go代码的分离，并且当我们更新HTML布局时也不需要修改代码。首先我们需要加入 `html/template` 包：
```go
import (
	"html/template"
	"io/ioutil"
	"net/http"
)
```

创建一个html模板文件edit.html，如下：
```html
<h1>Editing {{.Title}}</h1>

<form action="/save/{{.Title}}" method="POST">
<div><textarea name="body" rows="20" cols="80">{{printf "%s" .Body}}</textarea></div>
<div><input type="submit" value="Save"></div>
</form>
```

修改 editHandler 使用 template：
```go
func editHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/edit/"):]
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    t, _ := template.ParseFiles("edit.html")
    t.Execute(w, p)
}
```

函数 `template.ParseFiles` 将会读取 `edit.html` 文件的内容并且返回 `*template.Template`。

方法 t.Execute 执行模板, 根据传入的 p 生成最终的HTML文件并写入到 http.ResponseWriter。 模板文件中的 .Title 和 .Body 表示 p.Title 和 p.Body.

模板指令用双或括弧包裹`{{}}` ， 指令`printf "%s" .Body`是一个函数，该函数将`.Body`数据转换为字符串并输出。html/template 还确保输出安全和正确的HTML数据， 例如，它会自动的将大于号（>）替换为 `&gt` ，防止跨站脚本攻击。

接下来创建`view.html`模板文件用来显示页面：
```html
<h1>{{.Title}}</h1>

<p>[<a href="/edit/{{.Title}}">edit</a>]</p>

<div>{{printf "%s" .Body}}</div>
```

修改`viewHandler` :
```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, _ := loadPage(title)
    t, _ := template.ParseFiles("view.html")
    t.Execute(w, p)
}
```

观察 `viewHandler` 和 `editHandler` 函数的模板处理逻辑基本一样，因此我们可以将模板处理拿出来单独封装一个模板函数：
```go
func renderTemplate(w http.ResponseWriter, tmpl string, p *Page) {
  t, _ := template.ParseFiles(tmpl + ".html")
   t.Execute(w, p)
}
```

修改`viewHandler` 和 `editHandler`使用`renderTemplate`函数：
```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/view/"):]
    p, err := loadPage(title)
    if err != nil {
        http.Redirect(w, r, "/edit/"+title, http.StatusFound) // 找不到文件时重定位到编辑页面创建新的页面
        return
    }
    renderTemplate(w, "view", p)
}

func editHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/edit/"):]
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    renderTemplate(w, "edit", p)
}
```

**2.6 ERROR处理**

上面的代码的几个地方忽略了错误处理，在实际开发中这是绝对不允许的，因为当错误发生时会导致代码出现异常的行为。好的解决办法是接受函数返回的error并将错误信息发送给用户。这样当错误错误发生时可以通知用户。

首先在`renderTemplate`函数中加入错误处理：
```go
func renderTemplate(w http.ResponseWriter, tmpl string, p *Page) {
  t, err := template.ParseFiles(tmpl + ".html")
  if err != nil {
      http.Error(w, err.Error(), http.StatusInternalServerError)
      return
  }
  err = t.Execute(w, p)
  if err != nil {
      http.Error(w, err.Error(), http.StatusInternalServerError)
  }
}
```

接下来更新`saveHandler`：
```go
func saveHandler(w http.ResponseWriter, r *http.Request) {
    title := r.URL.Path[len("/save/"):]
    body := r.FormValue("body")
    p := &Page{Title: title, Body: []byte(body)}
    err := p.save()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Redirect(w, r, "/view/"+title, http.StatusFound)
}
```

p.save()返回的任何错误都将会报告给用户。

**2.7 模板缓存(Template caching)**

目前为止，代码中有一个影响效率的地方： `renderTemplate` 函数。当我们每次调用 `renderTemplate` 函数时都会执行一次`template.ParseFiles`函数来解析模板文件。一个更好的方式是在代码初始化的地方执行一次`template.ParseFiles`函数把所有的模板文件解析到 `*Template` 变量中，然后我们用`ExecuteTemplate`方法来渲染一个指定的模板。

首先，我们创建一个全局的 templates 变量：
```go
var templates = template.Must(template.ParseFiles("edit.html", "view.html"))
```

接下来修改`renderTemplate`函数调用`ExecuteTemplate`方法来执行指定的模板：
```go
func renderTemplate(w http.ResponseWriter, tmpl string, p *Page) {
    err := templates.ExecuteTemplate(w, tmpl+".html", p)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
}
```

**2.8 Validation**

web服务现在还有一个安全问题：用户能够使用任意的url访问服务，这将导致一个问题，如：“/view/test1/test2” ，此时文件名就会提取成 “test1/test2”，该文件名显然不符合文件的命名规范。因此为了避免该错误的发生，我们需要写一个函数来验证url中的`title`是否满足规范，验证方式采用正则表达式。

首先引入`regexp` 包，然后创建全局变量来保存我们的正则表达式：
```go
var validPath = regexp.MustCompile("^/(edit|save|view)/([a-zA-Z0-9]+)$")
```

写一个函数使用`validPath`来提取正确的页面`title`：
```go
func getTitle(w http.ResponseWriter, r *http.Request) (string, error) {
    m := validPath.FindStringSubmatch(r.URL.Path)
    if m == nil {
        http.NotFound(w, r)
        return "", errors.New("Invalid Page Title")
    }
    return m[2], nil // The title is the second subexpression.
}
```

接着我们更新每个handler来使用`getTitle`：
```go
func viewHandler(w http.ResponseWriter, r *http.Request) {
    title, err := getTitle(w, r)
    if err != nil {
        return
    }
    p, err := loadPage(title)
    if err != nil {
        http.Redirect(w, r, "/edit/"+title, http.StatusFound)
        return
    }
    renderTemplate(w, "view", p)
}


func editHandler(w http.ResponseWriter, r *http.Request) {
    title, err := getTitle(w, r)
    if err != nil {
        return
    }
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    renderTemplate(w, "edit", p)
}


func saveHandler(w http.ResponseWriter, r *http.Request) {
    title, err := getTitle(w, r)
    if err != nil {
        return
    }
    body := r.FormValue("body")
    p := &Page{Title: title, Body: []byte(body)}
    err = p.save()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Redirect(w, r, "/view/"+title, http.StatusFound)
}
```

**2.9 引入函数变量和闭包**

在每个handler处理函数进行错误判断会造成大量的代码重复。我们可以将每个handler放到一个执行错误检查和验证的函数中，这样在执行handler之前会首先进行错误验证，若验证通过再去执行 handler ，否则直接返回客户端错误。

首先，我们重写每个 handler 的定义：
```go
func viewHandler(w http.ResponseWriter, r *http.Request, title string)
func editHandler(w http.ResponseWriter, r *http.Request, title string)
func saveHandler(w http.ResponseWriter, r *http.Request, title string)
```

现在我们定义一个 `wrapper function` 来接收一个上面定义的函数类型，并且返回一个类型为`http.HandlerFunc` 的函数：
```go
func makeHandler(fn func (http.ResponseWriter, *http.Request, string)) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// Here we will extract the page title from the Request,
		// and call the provided handler 'fn'
	}
}
```

现在可以将`getTitle`的处理逻辑放到`makeHandler`中：
```go
func makeHandler(fn func(http.ResponseWriter, *http.Request, string)) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        m := validPath.FindStringSubmatch(r.URL.Path)
        if m == nil {
            http.NotFound(w, r)
            return
        }
        fn(w, r, m[2])
    }
}
```

现在，我们可以包裹 handler functions 使用 makeHandler：
```go
func main() {
    http.HandleFunc("/view/", makeHandler(viewHandler))
    http.HandleFunc("/edit/", makeHandler(editHandler))
    http.HandleFunc("/save/", makeHandler(saveHandler))

    http.ListenAndServe(":8080", nil)
}
```

最后我们将每个 haldler 函数中的 `getTitle` 移除:
```go
func viewHandler(w http.ResponseWriter, r *http.Request, title string) {
    p, err := loadPage(title)
    if err != nil {
        http.Redirect(w, r, "/edit/"+title, http.StatusFound)
        return
    }
    renderTemplate(w, "view", p)
}

func editHandler(w http.ResponseWriter, r *http.Request, title string) {
    p, err := loadPage(title)
    if err != nil {
        p = &Page{Title: title}
    }
    renderTemplate(w, "edit", p)
}

func saveHandler(w http.ResponseWriter, r *http.Request, title string) {
    body := r.FormValue("body")
    p := &Page{Title: title, Body: []byte(body)}
    err := p.save()
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    http.Redirect(w, r, "/view/"+title, http.StatusFound)
}
```

最后，一个功能简单的web服务就做完了！
