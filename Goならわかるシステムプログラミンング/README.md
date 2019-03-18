# Goで覗くシステムプログラミングの世界

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

---

# 低レベルアクセスへの入り口（1）：io.Writer
### io.WriterはOSが持つファイルのシステムコールの相似形
- `syscall.Write`は、GoからシステムコールのWrite関数を呼び出すのに使う関数である。OSでは、このWriteをファイルディスクリプタに対して呼ぶ。
- ファイルディスクリプタに対応するのはファイルだけでなく、標準入出力、ソケットなどの本来ファイルではないものにもファイルディスクリプタが割り当てられていて、どれもファイルと同じようにアクセスが可能である。
- POSIX系OSでは、可能な限り様々なものがファイルとして抽象化されている。しかし、Windowsだとソケットはファイルとして扱えなかったり、ファイル出力とコンソール出力でAPIが違っていたりとシステムによる違いも少なからずあるので、Go言語ではOSによるAPIの差異を`io.Writer`にて吸収している。

### io.Writerはインターフェース
`File.Write`の定義は以下の通りである、
```go
func (f *File) Write(b []byte) (n int, err error) {
	if err := f.checkValid("write"); err != nil {
		return 0, err
	}
	n, e := f.write(b)
	...
}
```
- これは、__書き込むバイト列`b`を受け取って書き込んだバイト数`n`とエラーが起きた場合の`error`型`err`を返す__ことがわかる。
- POSIX系OSでは様々なものがファイルとして抽象化されているので__書き込むバイト列`b`を受け取って書き込んだバイト数`n`とエラーが起きた場合の`error`型`err`を返す__という流れは通常のファイルに限らず色々なもので適用できそうである。Go言語では、その場合に使える仕組みとしてインターフェースという型が用意されている。
```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

### io.Writerを使う構造体の例
- ファイル出力
- 画面出力
- 書かれた内容を記憶しておくバッファ
- インターネットアクセス
