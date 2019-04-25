- deadcode 
Finds unused code.
```bash
$ find . -type d -not -path "./vendor/*" | xargs deadcode
```

- dingo-hunter
Static analyser for finding deadlocks in Go.
- dupl
Reports potentially duplicated code.

- gotype
```bash
$ go get -u golang.org/x/tools/cmd/gotype
```
```bash
find . -name "*.go" -not -path "./vendor/*" -not -path ".git/*" -print0 | xargs -0 gotype -a
```

- misspell
misspell用来拼写检查，对国内英语不太熟练的同学很有帮助
```bash
go get -u github.com/client9/misspell
```

```bash
$ find . -type f -not -path "./vendor/*" -print0 | xargs -0 misspell
```

- staticcheck
```bash
$ go get -u honnef.co/go/staticcheck/cmd/staticcheck
```
```bash	
$ staticcheck $(glide nv)
```
- gosimple
gosimple 提供信息，帮助你了解哪些代码可以简化。
```
$ go get -u honnef.co/go/simple/cmd/gosimple
```
```bash
$ gosimple $(glide nv)
```
- goconst
```bash
$ go get -u github.com/jgautheron/goconst/cmd/goconst
```
```bash
$ goconst ./… | grep -v vendor
```
- gocyclo
Gocyclo calculates cyclomatic complexities of functions in Go source code.
用来检查函数的复杂度,函数的基本复杂度是 1，然后每有一个 'if'，'for'，'case'，'&&' 或 '||' 则 +1
https://github.com/fzipp/gocyclo
```bash
gocyclo -over 10 $(ls -d */ | grep -v vendor)
gocyclo -avg  $(ls -d */ | grep -v vendor)
gocyclo -top 10  $(ls -d */ | grep -v vendor)
```

#### Install all tools
```bash
go get -u github.com/fzipp/gocyclo

```