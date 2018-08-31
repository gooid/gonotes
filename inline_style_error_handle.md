
[go2draft-error-handling-overview](https://go.googlesource.com/proposal/+/master/design/go2draft-error-handling-overview.md) 

关于 `check/handle` 是一个很好的方向，但 `handle err` 这个`err`会有困惑，并且对于有多错误返回的函数`func foo() (err1, err2 error) `来说`check`是对err1或err2会有歧义。

建议可在目前草案的基础上做小的调整，handle 改成类似C/C++的inline。
以实例来说明：

原草案代码：
```go
func CopyFile(src, dst string) error {
	handle err {
		return fmt.Errorf("copy %s %s: %v", src, dst, err)
	}

	r := check os.Open(src)
	defer r.Close()

	w := check os.Create(dst)
	handle err {
		w.Close()
		os.Remove(dst) // (only if a check fails)
	}

	check io.Copy(w, r)
	check w.Close()
	return nil
}
```

建议方案：
```go
func CopyFile(src, dst string) error {
	errI := inline(err error) {
        	if err != nil {
            		return fmt.Errorf("copy %s %s: %v", src, dst, err)
        	}
	}

	r, errI := os.Open(src)
	defer r.Close()

	w, errI := os.Create(dst)
	errI = inline(err error) {
        	if err != nil {
		    	w.Close()
            		os.Remove(dst) // (only if a check fails)
        	}
	}

	_, errI = io.Copy(w, r)
	errI = w.Close()
	return nil
}
```

对inline类型的变量赋值，相当于执行他的代码，要赋与的值则为其参数，象上面的代码相当于执行了现在的代码：
```go
func CopyFile(src, dst string) error {
	r, errI := os.Open(src); /* errI */ { var err error = errI; if err != nil { return fmt.Errorf("copy %s %s: %v", src, dst, err) } }
	defer r.Close()

	w, errI := os.Create(dst); /* errI */ { var err error = errI; if err != nil { return fmt.Errorf("copy %s %s: %v", src, dst, err) } }
	
	_, errI = io.Copy(w, r); /* errI */ { var err error = errI; if err != nil { w.Close(); os.Remove(dst) } }
	errI = w.Close(); /* errI */ { var err error = errI; if err != nil { w.Close(); os.Remove(dst) } }
	return nil
}
```

自然，它也是一种通用的机制，不仅仅用来处理error。比如：
```go
func ProcessStat(...) {
    statI := inline(stat Stat) {
        swith stat {
            ...
        }
    }

    errI := inline(err error) { ... }

    statI, errI := readStat1()
    ...
    statI, errI := readStat2()
}

```

对于多错误返回的情况也可清晰的处理：
```go
func foo() (err1, err2 error) {...}

func CallFoo() {
    errA := inline(err error) {...}
    errB := inline(err error) {...}

    errA, errB = foo()
}
```

对于多个返回值都用inline方式时，按惯例遵循从右到左执行。
因为这是个通用的方案，理论上可定义一个全局inline，但可能会导致一些风险(隐隐觉得有些不安^_^), 可限制它在func内有使用。
