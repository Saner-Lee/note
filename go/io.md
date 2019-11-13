# io

`io`包作为`golang`中最基本的库，对外暴露的方法的参数大多封装为`interface`，更加具有通用性，其中也有值得学习的地方。

```go
const (
	SeekStart   = 0 // seek relative to the origin of the file
	SeekCurrent = 1 // seek relative to the current offset
	SeekEnd     = 2 // seek relative to the end
)
```

这个技巧大家平常应该都会用到，将一些函数参数的意义定义为常量，通过常量名或者加一些注释体现其作用，避免在程序中出现魔数。

```go
// ErrShortWrite means that a write accepted fewer bytes than requested
// but failed to return an explicit error.
var ErrShortWrite = errors.New("short write")

// ErrShortBuffer means that a read required a longer buffer than was provided.
var ErrShortBuffer = errors.New("short buffer")

// EOF is the error returned by Read when no more input is available.
// Functions should return EOF only to signal a graceful end of input.
// If the EOF occurs unexpectedly in a structured data stream,
// the appropriate error is either ErrUnexpectedEOF or some other error
// giving more detail.
var EOF = errors.New("EOF")

// ErrUnexpectedEOF means that EOF was encountered in the
// middle of reading a fixed-size block or data structure.
var ErrUnexpectedEOF = errors.New("unexpected EOF")

// ErrNoProgress is returned by some clients of an io.Reader when
// many calls to Read have failed to return any data or error,
// usually the sign of a broken io.Reader implementation.
var ErrNoProgress = errors.New("multiple Read calls return no data or error")
```

1. 库中连续定义了多个包级的错误变量，使用库的时候可以比较错误，而非根据错误信息中是否包含一些字符串判断是否出错。
2. 并未使用`var (statment1;statment2)`（分号代表逻辑行结束，这里考虑文章格式没有展成多行，所以使用分号），而是每一个变量都使用`var`单独定义，这里主要是为了添加注释，错误是程序中很重要的一部分，因此有必要对这些变量加注释说明发生的场景。

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

type ReadWriter interface {
    Reader
    Writer
}

type ReadCloser interface {
    Reader
    Closer
}

type WriteCloser interface {
    Writer
    Closer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

这里并没有列举出包中所有的接口类型，但是列举出的接口类型足以说明使用接口的一种规范，接口类型的功能应该最小化，使用小接口组合出复杂的接口。其实正是职责最小化的一种体现，比如`func Copy(dst Writer, src Reader) (written int64, err error) `，参数类型应该是能满足当前场景工作的类型中的最小类型，因此将参数类型限定为小接口类型，虽然实参可以给一个复杂接口（比如`ReadWriteCloser`）的实例，但是在`Copy`中只把它当做`Reader`或者`Writer`，只使用需要的功能，不需要的功能不应该出现。

下面就是通过源码进一步认识一下为什么`io`包定义很多小接口且将组合出各种比较复杂的接口。

```go
func copyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	// 如果src实现了WriteTo接口，直接使用WriteTo完成复制
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// 如果dst实现了ReadFrom接口，直接使用ReadFrom完成复制
	if rt, ok := dst.(ReaderFrom); ok {
		return rt.ReadFrom(src)
	}
    
        // 如果外部提供了buffer，则复用，这样外部可以控制每次拷贝的大小
        // 分配一个拷贝用的buffer
        // 默认是32k，如果src是LimitedReader，并且大小小于32k，则改变默认大小
	if buf == nil {
		size := 32 * 1024
		if l, ok := src.(*LimitedReader); ok && int64(size) > l.N {
			if l.N < 1 {
				size = 1
			} else {
				size = int(l.N)
			}
		}
		buf = make([]byte, size)
	}
    
        // 完成src到dst的拷贝
	for {
		nr, er := src.Read(buf)
		if nr > 0 {
			nw, ew := dst.Write(buf[0:nr])
			if nw > 0 {
				written += int64(nw)
			}
			if ew != nil {
				err = ew
				break
			}
			if nr != nw {
				err = ErrShortWrite
				break
			}
		}
        
                // 这里EOF并不会视为错误，因此发生EOF错误时，err为nil
		if er != nil {
			if er != EOF {
				err = er
			}
			break
		}
	}
	return written, err
}

// 直接使用copyBuffer，err == nil视为成功，没有err == EOF的情况
func Copy(dst Writer, src Reader) (written int64, err error) {
	return copyBuffer(dst, src, nil)
}

// 不合法的buf提前panic
// 成功的情况同Copy
func CopyBuffer(dst Writer, src Reader, buf []byte) (written int64, err error) {
	if buf != nil && len(buf) == 0 {
		panic("empty buffer in io.CopyBuffer")
	}
	return copyBuffer(dst, src, buf)
}

// 内部构造一个LimitedReader，这样可以复用copyBuffer
// 这里可能会出现EOF错误，因为指定的n可能超过了Reader的容量
func CopyN(dst Writer, src Reader, n int64) (written int64, err error) {
	written, err = Copy(dst, LimitReader(src, n))
	if written == n {
		return n, nil
	}
	if written < n && err == nil {
		// src stopped early; must have been EOF.
		err = EOF
	}
	return
}
```

在实现`copyBuffer`时，考虑了几种接口，比如`WriteTo`和`ReadFrom`可以使用已有的功能完成拷贝，又比如可以通过使用`LimitedReader`可以限制拷贝的数量。

下面再来两个包中不是特别常用的函数，

```go
// n可能大于min，这取决于传递的是什么样的参数
// len(buf) == min未发生错误时n == min
// len(buf) > min未发生错误时，n > min
func ReadAtLeast(r Reader, buf []byte, min int) (n int, err error) {
	// 参数错误
        if len(buf) < min {
		return 0, ErrShortBuffer
	}
    
        // 未完成读取任务时继续读取
	for n < min && err == nil {
		var nn int
		nn, err = r.Read(buf[n:])
		n += nn
	}
	if n >= min {
		err = nil                       // 成功时err为nil
	} else if n > 0 && err == EOF {
		err = ErrUnexpectedEOF          // 读取到末尾但数量不满足，err为ErrUnexcptedEOF
	}
	return
}

// 将buf读满
// 使用len(buf) == min未发生错误时n == min的特性完成功能
func ReadFull(r Reader, buf []byte) (n int, err error) {
	return ReadAtLeast(r, buf, len(buf))
}
```

最后，还有一点在注释方面值得学习的地方，也是当前`golang`的规范：

- 方法或者函数的注释，都以方法名或者函数名开始

- 包的包注释应当使用c注释风格，且应该出现在第一个文件的顶部
