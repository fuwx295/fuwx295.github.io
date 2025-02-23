+++
author = "GreyWind"
title = "The Go Programming Language 课后练习 Ch.1"
date = "2023-07-27"
description = "The Go Programming Language 课后练习 Ch.1"
tags = [
    "the-go-programming-language",
]
image="ab638d52-5f92-4e29-955e-064deb1c4728.jpeg"
+++

## 前言

第一章为Tutorial（教程），通过8个部分的示例代码带领读者熟悉Go语言的语法和规则。第一章课后练习比较基础，熟悉基本语法规则即可完成。

本章所有示例代码链接 [Ch1](https://github.com/adonovan/gopl.io/tree/master/ch1) (有需求自取，就不给出示例代码了。。。)

## 课后练习

### Command-Line Arguments

示例代码是一个打印命令行参数的程序，课后练习如下

> **练习 1.1**： 修改 `echo` 程序，使其能够打印 `os.Args[0]`，即被执行命令本身的名字
>
> **练习 1.2**： 修改 `echo` 程序，使其打印每个参数的索引和值，每个一行。
>
> **练习 1.3**： 做实验测量潜在低效的版本和使用了`strings.Join`的版本的运行时间差异。（[1.6节](https://golang-china.github.io/gopl-zh/ch1/ch1-06.html)讲解了部分`time`包，[11.4节](https://golang-china.github.io/gopl-zh/ch11/ch11-04.html)展示了如何写标准测试程序，以得到系统性的性能评测。）

思路

这三个练习还是非常基础的，利用`for range`遍历`os.Args[0:]`切片即可，`range`会返回**元素**和该元素对应的**下标**（从0开始）。示例代码中使用字符串拼接的方式来获取命令行参数结果，但其实这是一种效率非常低的方法，因此另一个示例代码用`stirngs.Join`方法拼接字符串，这是一种更高效的方法（原因参考原文）。为了测试两种方式的效率，使用`time`包中的`Now()`和`Since()`方法即可获得程序运行时间，比较运行时间即可测评性能。

```go
func main() {
    for i, arg := range os.Args[0:] {
        fmt.Printf("index = %d, value = %s\n", i, arg)
    }
}
```

```go
func main() {
    start := time.Now()

    s, sep := "", ""
    for _, arg := range os.Args[0:] {
        s += sep + arg
        sep = " "
    }
    fmt.Println(s)

    // strings.Join
    // fmt.Println(strings.Join(os.Args[0:], " "))

    total := time.Since(start)
    fmt.Printf("runing time = %s\n", total)
}
```

### Finding Duplicate Lines

示例代码是一个读取标准输入流或者命令行参数中的文件, 并找出重复行的程序。课后练习如下

> **练习 1.4**： 修改 dup2，出现重复的行时打印文件名称。

思路

原代码通过维护一个`map`来记录每行的内容和出现次数，因此需维护另一个`map`来记录每行内容和出现该内容的`file`数组。最后在输出重复行时，将`file`数组一同输出即可。

```go
// if target in strs
func contains(strs []string, target string) bool {
    for _, str := range strs {
        if str == target {
            return true
        }
    return false
}

// find duplicate line
func countLines(f *os.File, counts map[string]int, filename map[string][]string) {
    intput := bufio.NewScanner(f)
    for input.Scan() {
        str := input.Text()
        counts[str]++
        if filename == nil {
            filename[str] = make([]string, 0)
        }
        if !contains(filename[str], f.Name()) {
            filename[str] = append(filename[str], f.Name())
        }
}

func main() {
    counts := make(map[string]int)
    filename := make(map[string][]string)
    files := os.Args[1:]
    if len(files) == 0 {
        countLines(os.Stdin, counts, filename)
    } else {
        for _, arg := range files {
            f, err := os.Open(arg)
            if err != nil {
                fmt.Fprintf(os.Stderr, "dup2: %v\n", err)
                continue
            }
            // dup line
            countLines(f, counts, filename)
            f.Close()
        }
    }
    // output info
    for line, n := range counts {
        if n > 1 {
            fmt.Printf("%d\t%s\t%v\n", n, line, filename[line])
        }
    }
}
```

### Animated GIFs

示例代码是一个利用sin函数和cos函数打印利萨如图形（Lissajous figures）的程序。课后练习如下

> **练习 1.5**： 修改前面的Lissajous程序里的调色板，由黑色改为绿色。我们可以用`color.RGBA{0xRR, 0xGG, 0xBB, 0xff}`来得到`#RRGGBB`这个色值，三个十六进制的字符串分别代表红、绿、蓝像素。
>
> **练习 1.6**： 修改Lissajous程序，修改其调色板来生成更丰富的颜色，然后修改SetColorIndex的第三个参数，看看显示结果吧。

思路

`palette`数组为图形调色板，`SetColorIndex`函数第三个参数对应调色板颜色的**下标**。将七彩色放入调色板，每次运行后改变颜色即可得到`RGB`GIF。

![Lissajous](af5138fe-143e-4add-a186-e9b30a741534.gif "Lissajous")

```go
var palette = []color.Color{
	//color.Black,
	color.White,
	color.RGBA{0xff, 0x00, 0x00, 0xff},
	color.RGBA{0xff, 0x7d, 0x00, 0xff},
	color.RGBA{0xff, 0xff, 0x00, 0xff},
	color.RGBA{0x00, 0xff, 0x00, 0xff},
	color.RGBA{0x00, 0x00, 0xff, 0xff},
	color.RGBA{0x00, 0xff, 0xff, 0xff},
	color.RGBA{0xff, 0x00, 0xff, 0xff},
}

func lissajous(out io.Writer) {
	const (
		cycles  = 5
		res     = 0.001
		size    = 100
		nframes = 64
		delay   = 8
	)
	freq := rand.Float64() * 3.0
	anim := gif.GIF{LoopCount: nframes}
	phase := 0.0
	colorIndex := 1
	for i := 0; i < nframes; i++ {
		rect := image.Rect(0, 0, 2*size+1, 2*size+1)
		img := image.NewPaletted(rect, palette)
		for t := 0.0; t < cycles*2*math.Pi; t += res {
			x := math.Sin(t)
			y := math.Sin(t*freq + phase)
			img.SetColorIndex(size+int(x*size+0.5), size+int(y*size+0.5), uint8(colorIndex))
		}
		colorIndex = (colorIndex % 7) + 1
		phase += 0.1
		anim.Delay = append(anim.Delay, delay)
		anim.Image = append(anim.Image, img)
	}
	gif.EncodeAll(out, &anim)

}
```

### Fetching a URL

示例代码是一个获取对应的url，发送http请求，并将其源文本打印出来的程序，课后练习如下

> **练习 1.7**： 函数调用io.Copy(dst, src)会从src中读取内容，并将读到的结果写入到dst中，使用这个函数替代掉例子中的ioutil.ReadAll来拷贝响应结构体到os.Stdout，避免申请一个缓冲区（例子中的b）来存储。记得处理io.Copy返回结果中的错误。
>
> **练习 1.8**： 修改fetch这个范例，如果输入的url参数没有 http:// 前缀的话，为这个url加上该前缀。你可能会用到strings.HasPrefix这个函数。
>
> **练习 1.9**： 修改fetch打印出HTTP协议的状态码，可以从resp.Status变量得到该状态码。

思路

这三个练习较为基础，用`io.Copy`代替**ioutill.ReadAll**，将返回信息输出到`stdout`中。通过`strings.HasPrefix`来判断`url`是否有`http://`前缀，没有就加上。最后打印`resp.Status`即可。

```go
func main() {
	for _, url := range os.Args[1:] {
		if !strings.HasPrefix(url, "http://") {
			url = "http://" + url
		}
		resp, err := http.Get(url)
		if err != nil {
			fmt.Fprint(os.Stderr, "fetch: %v\n", err)
			os.Exit(1)
		}
		//b, err := ioutil.ReadAll(resp.Body)
		fmt.Printf("http status: %s\n", resp.Status)
		b, err := io.Copy(os.Stdout, resp.Body)
		resp.Body.Close()
		if err != nil {
			fmt.Fprint(os.Stderr, "fetch: iocopy %s: %v\n", url, err)
			os.Exit(1)
		}
		fmt.Printf("%d\n", b)
	}
}
```

### Fetching URLS Concurrently

示例代码和上一小节程序所要做的工作基本一致，但`fetchall`的特别之处在于它会同时去获取所有的URL，所以这个程序的总执行时间不会超过执行时间最长的那一个任务，前面程序执行时间则是所有任务执行时间之和。课后练习如下

> **练习 1.10**： 找一个数据量比较大的网站，用本小节中的程序调研网站的缓存策略，对每个URL执行两遍请求，查看两次时间是否有较大的差别，并且每次获取到的响应内容是否一致，修改本节中的程序，将响应结果输出，以便于进行对比。
>
> **练习 1.11**： 在fetchall中尝试使用长一些的参数列表，比如使用在[alexa.com](https://alexa.com/)的上百万网站里排名靠前的。如果一个网站没有回应，程序将采取怎样的行为？（Section8.9 描述了在这种情况下的应对机制）。

思路

这里涉及的内容为计算机网络内容，个人无法给出规范答案，因此由ChatGPT作答

1.10

> 根据网站的缓存策略，对每个URL执行两遍请求，两次执行时间可能会有较大的差别，这取决于缓存的设置和实际的网络环境。当涉及到缓存的情况时，以下两种情况是可能的：
>
> 第一次请求：如果该URL的响应之前没有被缓存，服务器将会生成响应并将其返回给客户端。在这种情况下，第一次请求可能会比较慢，因为服务器需要处理请求并生成响应。
>
> 第二次请求：如果该URL的响应被缓存了，服务器会直接返回缓存的响应，而无需再次处理请求。在这种情况下，第二次请求通常会比较快，因为省去了服务器重新生成响应的步骤。
>
> 因此，第一次请求通常比第二次请求慢，但具体的差别取决于缓存策略、缓存的生存时间以及服务器和网络的负载等因素。
>
> 关于响应内容的一致性，如果两次请求之间没有发生数据变化，并且没有其他中间因素影响响应内容，那么两次获取到的响应内容应该是一致的。这是因为缓存是根据URL或其他标识来存储响应的，如果请求的标识相同，那么缓存会返回相同的响应内容。
> 然而，需要注意以下几点：
>
> 如果两次请求之间的数据发生了变化（例如，某些数据被更新了），则两次响应的内容可能会不一致，因为第二次请求获取到的是最新的数据。
>
> 如果网站的缓存策略是根据时间来进行刷新，那么两次请求之间的时间间隔较长，第二次请求可能会从服务器上获取到更新后的响应内容。
>
> 在多服务器或负载均衡环境中，两次请求可能被分发到不同的服务器上，导致两次响应内容不一致，这取决于服务器之间是否共享缓存。
>
> 综上所述，根据网站的缓存策略，两次请求的执行时间可能有较大的差别，并且每次获取到的响应内容在一些特定情况下可能会不一致。但在正常情况下，如果两次请求之间没有发生变化并且没有其他中间因素干扰，响应内容应该是一致的。

1.11

> 在`fetchall`代码中，如果尝试使用长一些的参数列表，例如使用在 Alexa 排名靠前的上百万网站作为URL列表，当其中某个网站没有回应时，程序的行为将根据并发请求的方式来决定。
>
> 根据目前给出的代码，`fetchall`是使用一个简单的循环来依次请求每个URL的，因此程序会按照顺序一个一个地发送请求。在这种情况下，如果一个网站没有回应，程序会出现阻塞等待的情况，直到该请求超时或获得响应。
>
> 具体表现如下：
>
> 阻塞等待：当程序发起请求并没有获得响应时，程序会一直等待，直到超过了设置的请求超时时间。这可能会导致程序执行时间较长，尤其在长列表中有多个未响应的网站时。
>
> 超时报错：如果请求超时，http.Get方法会返回一个错误，指示请求超时。在代码中，该错误会被捕获，并在标准错误输出中打印出来。随后，程序继续执行下一个请求。
>
> 值得注意的是，超时时间的设置在当前给出的代码中并未体现。对于大规模请求或长列表，设置合理的超时时间是非常重要的。如果没有设置超时时间，程序在遇到未响应的请求时会一直等待，可能会导致整个程序执行时间过长或出现假死状态。
>
> 为了优化fetchall的行为，可以考虑采用并发的方式发送请求，从而避免阻塞等待问题。可以使用 Goroutines 和通道来实现并发请求，从而提高程序的效率。在并发的情况下，当某个请求超时或出现问题时，其他请求仍可以继续执行，不会受到阻塞。这样可以更好地利用系统资源，并加快整体请求的处理速度。
>
> 以下是一个简单的示例代码，展示了如何使用 Goroutines 和通道来并发地发送请求：

```go
package main

import (
    "fmt"
    "net/http"
    "io/ioutil"
    "os"
    "time"
)

func fetch(url string, ch chan<- string) {
    start := time.Now()
    resp, err := http.Get(url)
    if err != nil {
        ch <- fmt.Sprintf("fetch error: %s %v", url, err)
        return
    }
    defer resp.Body.Close()

    b, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        ch <- fmt.Sprintf("read error: %s %v", url, err)
        return
    }

    ch <- fmt.Sprintf("%.2fs %s", time.Since(start).Seconds(), b)
}

func main() {
    start := time.Now()
    ch := make(chan string)

    for _, url := range os.Args[1:] {
        go fetch(url, ch)
    }

    for range os.Args[1:] {
        fmt.Println(<-ch)
    }

    fmt.Printf("Total time: %.2fs\n", time.Since(start).Seconds())
}
```

> 在这个示例中，fetch函数被并发地调用，并使用通道ch来收集每个请求的结果。当一个请求完成时，它会将结果发送到通道中，然后主函数从通道中接收结果并打印。这样，即使其中某个请求没有回应，其他请求仍然可以继续执行，不会出现阻塞等待的情况。

### A WebServer

示例代码为一个简单的Web服务器程序。由于go标准库实现了大量的工作，我们只需写少数代码即可实现服务器。课后练习如下

> 练习 1.12： 修改Lissajour服务，从URL读取变量，比如你可以访问 http://localhost:8000/?cycles=20 这个URL，这样访问可以将程序里的cycles默认的5修改为20。字符串转换为数字可以调用strconv.Atoi函数。你可以在godoc里查看strconv.Atoi的详细说明。

思路

解析`url`中的`cycles`参数，然后改变`lissajour`函数，将`cycles`参数传入其中，就可以变化`gif`图形

```go
func main() {
	handler := func(w http.ResponseWriter, r *http.Request) {
		var cycles string
		for k, v := range r.Form {
			if k == "cycles" {
				cycles = v[0]
			}
		}
		cycles_int, err := strconv.Atoi(cycles)

		if err != nil {
			cycles_int = 5
		}

		lissajous(w, cycles_int)
	}
	http.HandleFunc("/", handler)
	log.Fatal(http.ListenAndServe("localhost:8000", nil))

}
```