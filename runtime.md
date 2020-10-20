# 获取cpu 核心数

```
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Println(runtime.NumCPU())
}

```

```
-> % go run main.go  
4
```

