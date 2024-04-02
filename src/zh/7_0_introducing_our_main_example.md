# 介绍我们的主要示例

现在我们终于到了书中我们将编写更多代码的部分。

Node事件循环是经过多年发展的复杂软件。我们将不得不大大简化事情。

我们将尝试实现对我们更好理解Node的重要部分，最重要的是使用它作为一个示例，我们可以利用前面章节的知识来创建真正有效的东西。

我们在这里的主要目标是探索异步概念，使用Node作为示例主要是为了有趣。

**我们想要编写类似于以下的代码：**

```rust, no_run
/// Think of this function as the javascript program you have written
fn javascript() {
    print("First call to read test.txt");
    Fs::read("test.txt", |result| {
        let text = result.into_string().unwrap();
        let len = text.len();
        print(format!("First count: {} characters.", len));

        print(r#"I want to create a "magic" number based on the text."#);
        Crypto::encrypt(text.len(), |result| {
            let n = result.into_int().unwrap();
            print(format!(r#""Encrypted" number is: {}"#, n));
        })
    });

    print("Registering immediate timeout 1");
    set_timeout(0, |_res| {
        print("Immediate1 timed out");
    });
    print("Registering immediate timeout 2");
    set_timeout(0, |_res| {
        print("Immediate2 timed out");
    });

    // let's read the file again and display the text
    print("Second call to read test.txt");
    Fs::read("test.txt", |result| {
        let text = result.into_string().unwrap();
        let len = text.len();
        print(format!("Second count: {} characters.", len));

        // aaand one more time but not in parallel.
        print("Third call to read test.txt");
        Fs::read("test.txt", |result| {
            let text = result.into_string().unwrap();
            print_content(&text, "file read");
        });
    });

    print("Registering a 3000 and a 500 ms timeout");
    set_timeout(3000, |_res| {
        print("3000ms timer timed out");
        set_timeout(500, |_res| {
            print("500ms timer(nested) timed out");
        });
    });

    print("Registering a 1000 ms timeout");
    set_timeout(1000, |_res| {
        print("SETTIMEOUT");
    });

    // `http_get_slow` let's us define a latency we want to simulate
    print("Registering http get request to google.com");
    Io::http_get_slow("http//www.google.com", 2000, |result| {
        let result = result.into_string().unwrap();
        print_content(result.trim(), "web call");
    });
}

fn main() {
    let rt = Runtime::new();
    rt.run(javascript);
}
```

> 我们将在代码的关键位置添加打印语句，这样我们就可以清楚地了解实际发生了什么，以及何时何地发生。

一开始你就会注意到我们在示例中编写的Rust代码有些奇怪。

该代码在执行异步操作时使用了回调，就像JavaScrip 一样，并且我们有像`Fs`或`Crypto`这样的“神奇”模块，可以像在Node中导入模块一样调用。

我们的代码主要是调用注册事件并存储回调以在事件准备就绪时运行的函数。

**这个例子就是`set_timeout`函数：**
```rust, no_run
set_timeout(0, |_res| {
    print("Immediate1 timed out");
});
```
我们实际上是在注册一个`timeout`事件的兴趣，当该事件发生时，我们希望运行回调`|_res| { print("Immediate1 timed out"); }`。

现在参数`_res`是传递给我们回调函数的参数。在JavaScript中，它会被省略，但由于我们使用的是类型化语言，我们创建了一个名为`Js`的类型。

`Js`是表示JavaScript类型的枚举。在`set_timeout`中，它是`Js::undefined`。在`Fs::read`中，它是`Js::String`，依此类推。

当我们运行此代码时，我们将得到类似以下的内容：

<video autoplay loop>
<source src="./images/example_run.mp4" type="video/mp4">
Can't display video.
</video>

这是我们输出的样子：
```
Thread: main	 First call to read test.txt
Thread: main	 Registering immediate timeout 1
Thread: main	 Registered timer event id: 2
Thread: main	 Registering immediate timeout 2
Thread: main	 Registered timer event id: 3
Thread: main	 Second call to read test.txt
Thread: main	 Registering a 3000 and a 500 ms timeout
Thread: main	 Registered timer event id: 5
Thread: main	 Registering a 1000 ms timeout
Thread: main	 Registered timer event id: 6
Thread: main	 Registering http get request to google.com
Thread: pool3	 recived a task of type: File read
Thread: pool2	 recived a task of type: File read
Thread: main	 Event with id: 7 registered.
Thread: main	 ===== TICK 1 =====
Thread: main	 Immediate1 timed out
Thread: main	 Immediate2 timed out
Thread: pool3	 finished running a task of type: File read.
Thread: pool2	 finished running a task of type: File read.
Thread: main	 First count: 39 characters.
Thread: main	 I want to create a "magic" number based on the text.
Thread: pool3	 recived a task of type: Encrypt
Thread: main	 ===== TICK 2 =====
Thread: main	 SETTIMEOUT
Thread: main	 Second count: 39 characters.
Thread: main	 Third call to read test.txt
Thread: main	 ===== TICK 3 =====
Thread: pool2	 recived a task of type: File read
Thread: pool3	 finished running a task of type: Encrypt.
Thread: main	 "Encrypted" number is: 63245986
Thread: main	 ===== TICK 4 =====
Thread: pool2	 finished running a task of type: File read.

===== THREAD main START CONTENT - FILE READ =====
Hello world! This is a text to encrypt!
... [Note: Abbreviated for display] ...
===== END CONTENT =====

Thread: main	 ===== TICK 5 =====
Thread: epoll	 epoll event 7 is ready

===== THREAD main START CONTENT - WEB CALL =====
HTTP/1.1 302 Found
Server: Cowboy
Location: http://http/www.google.com
... [Note: Abbreviated for display] ...
===== END CONTENT =====

Thread: main	 ===== TICK 6 =====
Thread: epoll	 epoll event timeout is ready
Thread: main	 ===== TICK 7 =====
Thread: main	 3000ms timer timed out
Thread: main	 Registered timer event id: 10
Thread: epoll	 epoll event timeout is ready
Thread: main	 ===== TICK 8 =====
Thread: main	 500ms timer(nested) timed out
Thread: pool0	 recived a task of type: Close
Thread: pool1	 recived a task of type: Close
Thread: pool2	 recived a task of type: Close
Thread: pool3	 recived a task of type: Close
Thread: epoll	 recieved event of type: Close
Thread: main	 FINISHED
```

不用担心，我们会解释一切，我只是想先解释一下我们想要达到的目标。
