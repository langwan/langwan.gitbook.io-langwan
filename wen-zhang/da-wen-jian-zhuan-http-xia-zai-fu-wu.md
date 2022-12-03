# 大文件转HTTP下载服务

### 单文件

任何文件都可以视频、音频、zip

#### 原生 go 的实现

```go
func TestDownloadFileByHttp(t *testing.T) {
	fp := "./samples/1.mp4"
	name := filepath.Base(fp)
	http.HandleFunc("/"+name, func(writer http.ResponseWriter, request *http.Request) {
		http.ServeFile(writer, request, fp)
	})
	err := http.ListenAndServe(":8100", nil)
	assert.NoError(t, err)
}
```

访问地址：

```
http://127.0.0.1:8100/1.mp4
```

#### Kratos 或者 Gin 实现

所有的http服务都包装了 http.HandleFunc 你一定可以找到一个response和request，套用即可。

```go
func TestDownloadFileByGin(t *testing.T) {
	fp := "./samples/1.mp4"
	name := filepath.Base(fp)
	g := gin.Default()

	g.GET("/"+name, func(context *gin.Context) {
		http.ServeFile(context.Writer, context.Request, fp)
	})
	g.StaticFile("/sf_"+name, fp)

	err := g.Run(":8100")
	err = http.ListenAndServe(":8100", nil)
	assert.NoError(t, err)
}
```

访问地址

```
http://127.0.0.1:8100/1.mp4
http://127.0.0.1:8100/sf_1.mp4
```

### 单文件数据流方式

```go
func TestDownloadFileStreamByHttp(t *testing.T) {
	fp := "./samples/1.mp4"
	name := filepath.Base(fp)
	http.HandleFunc("/"+name, func(writer http.ResponseWriter, request *http.Request) {
		f, _ := os.Open(fp)
		defer f.Close()
		info, _ := f.Stat()
		http.ServeContent(writer, request, name, info.ModTime(), f)
	})
	err := http.ListenAndServe(":8100", nil)
	assert.NoError(t, err)
}
```

### 下载整个目录

```go
func TestDownloadDirByGin(t *testing.T) {
	dir := "./samples"
	g := gin.Default()
	g.Static("/samples", dir)
	err := g.Run(":8100")
	err = http.ListenAndServe(":8100", nil)
	assert.NoError(t, err)
}
```

访问地址

```
http://127.0.0.1:8100/samples/1.mp4
```

### 自定义数据流，可用于文件加密

```go
type MyStreamer struct {
	File *os.File
}

func (m *MyStreamer) Read(p []byte) (n int, err error) {
	n, err = m.File.Read(p)
	fmt.Printf("my streamer reads %d, err = %v\n", n, err)
	return n, err
}

func (m *MyStreamer) Seek(offset int64, whence int) (int64, error) {
	return m.File.Seek(offset, whence)
}

func TestDownloadFileMyStreamByHttp(t *testing.T) {
	fp := "./samples/1.mp4"
	name := filepath.Base(fp)

	http.HandleFunc("/"+name, func(writer http.ResponseWriter, request *http.Request) {
		f, _ := os.Open(fp)
		defer f.Close()
		myStreamer := MyStreamer{File: f}
		info, _ := f.Stat()
		http.ServeContent(writer, request, name, info.ModTime(), &myStreamer)
	})
	err := http.ListenAndServe(":8100", nil)
	assert.NoError(t, err)
}
```

控制台输出

```
=== RUN   TestDownloadFileMyStreamByHttp
my streamer reads 512, err = <nil>
my streamer reads 32768, err = <nil>
my streamer reads 32768, err = <nil>
my streamer reads 32768, err = <nil>
my streamer reads 32768, err = <nil>
my streamer reads 32768, err = <nil>
my streamer reads 32768, err = <nil>
my streamer reads 32768, err = <nil>
my streamer reads 32768, err = <nil>
my streamer reads 32768, err = <nil>
my streamer reads 32768, err = <nil>
my streamer reads 32768, err = <nil>
my streamer reads 32768, err = <nil>
```

### 导入的代码头

```go
import (
	"fmt"
	"github.com/gin-gonic/gin"
	"github.com/stretchr/testify/assert"
	"net/http"
	"os"
	"path/filepath"
	"testing"
)
```

### 代码和视频

[https://github.com/langwan/chihuo/tree/main/go%E8%AF%AD%E8%A8%80/%E5%AE%9E%E7%8E%B0%E5%A4%A7%E6%96%87%E4%BB%B6%E7%9A%84HTTP%E4%B8%8B%E8%BD%BD](https://github.com/langwan/chihuo/tree/main/go%E8%AF%AD%E8%A8%80/%E5%AE%9E%E7%8E%B0%E5%A4%A7%E6%96%87%E4%BB%B6%E7%9A%84HTTP%E4%B8%8B%E8%BD%BD)

[https://www.bilibili.com/video/BV118411Y7B8/](https://www.bilibili.com/video/BV118411Y7B8/)
