
![](assets/Pasted%20image%2020230610203555.png)


Go 的文件读写方式很多，可以使用 strings、bufio、os、io 等包

多数对文件的操作，都是实现了 io 包中的 `io.Reader` 和 `io.Writer` 接口。


# IO

这个包不涉及具体的 IO 实现，定义的是最基本的 IO 接口语义

基础类型：
- `Reader`
- `Writer`
- `Closer`
- `ReaderAt`
- `WriterAt`
- ...

组合类型：
- `ReaderCloser`
- `WriterCloser`
- ...

进阶：
- `TeeReader`
- `LimitReader`
- ...


> Reader 接口的定义
 
```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

其中 Read 方法接收一个字节切片，返回值为 int、error，即所读的字节数和遇到的任何错误。
go 文档中提到，当读取的字节数大于0时遇到 error，返回的字节数为已经读取的字节数长度。当顺利读取完输入流后，应当返回 0 和 EOF。

> Writer 接口定义

```go
type Writer interface {
	Write(p []byte (n int, err error)
}
```

Write 将长度为 len(p) 个字节从 p 写到基础数据流中。它返回从 p 写入的字节数 n 和遇到的导致写入提前停止的任何错误。如果 Write 返回 n < len(p)，它必须返回一个非零的错误。Write 不能修改分片数据，即使是暂时的。

这里实现一个 Reader 以加深理解

```go
package main
import (
    "fmt"
    "io"
)
type customReader struct {
    src string // 读取的资源
    cur int    // 当前读取的位置
}

// 创建一个 custom Reader
func newCustomReader(src string) *customReader {
    return &customReader{src: src, cur: 0}
}

func filter(r byte) byte {
    if (r >= 'A' && r <= 'Z') || (r >= 'a' && r <= 'z') {
        return r
    }
    return 0
}

// 实现 read 方法
func (cR *customReader) Read(p []byte) (int, error) {
    // 当前位置大于等于 len（src），则说明读取到了结尾，返回 EOF
    if cR.cur >= len(cR.src) {
        return 0, io.EOF
    }
    rest_len := len(cR.src) - cR.cur // 剩余未读取的长度
    n, bound := 0, 0
    if rest_len >= len(p) {
        bound = len(p) // 剩余长度大于 []byte  大小，说明可将缓冲区填满
    } else {
        bound = rest_len
    }
    buf := make([]byte, bound)
    for n < bound {
        // 每次读一个字节，仅保留字母
        if char := filter(cR.src[cR.cur]); char != 0 {
            buf[n] = char
        }
        n++
        cR.cur++
    }

    // 将处理后的 buf 复制到 p 中
    copy(p, buf)
    return n, nil
}

func main() {
    reader := newCustomReader("Hello, do something here. !@#? exit.")
    p := make([]byte, 5)
    for {
        n, err := reader.Read(p)
        if err == io.EOF {
            fmt.Println()
            break
        }
        fmt.Print(string(p[:n]))
    }
}
```

输出:
Hellodosomethinghereexit

# os.File

同样也实现了 `io.Reader` 和 `io.Writer`,因此可在任何 io 上下文中使用

例如，bufio 与 os 一同使用

```go
func WriteTo_test() {
    f, err := os.OpenFile("test.md", os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()
    r := bufio.NewReader(strings.NewReader("the second sentence\n"))
    n, _ := r.WriteTo(f)
    fmt.Printf("strings size: %v\n", n)
}
```