- `$ go get -u github.com/derekparker/delve/cmd/dlv`で入れたやつでデバッガがつけられる(Goでデバッガ初めて使ったので知らなかった)
- `fmt.Pritln`の流れをなんとなく追った。
  + 実態としては `Fprintln` を呼んでいる。
  + その中では printerのstateを格納するための構造体`pp`を生成してバッファを用意し、`io.Writer`の`Write` メソッドを呼んでいる。
  + Writeメソッドでは、`File.write`を呼んでいて、そこで初めてOS毎に呼ばれるコードが異なる。
  + 下のは `go version go1.10.2 darwin/amd64`の時のやつ
  + `syscall.Write(fd.Sysfd, p[nn:max])`がシステムコールである

```go
// fd_unix.go
func (f *File) write(b []byte) (n int, err error) {
	n, err = f.pfd.Write(b)
	runtime.KeepAlive(f)
	return n, err
}

// f.pfd.Writeで呼ばれてるやつ
func (fd *FD) Write(p []byte) (int, error) {
  // -snip-
	for {
		max := len(p)
		if fd.IsStream && max-nn > maxRW {
			max = nn + maxRW
		}
		n, err := syscall.Write(fd.Sysfd, p[nn:max])
    		...
	}
  // -snip-
}
```
