---
layout: post
title: Rust Web 全栈开发教程
date: 2023-09-24 16:45:30.000000000 +09:00
categories: [Rust]
tags: [Rust]
---


# 课程主要内容
* 使用 Rust 开发一个全栈的 Web 应用，它包括：
1. `Web Service`
2. 服务器段渲染的 `Web App`
3. 客户端渲染的 `Web App（WebAssembly）`
4. 使用 Web 框架：`Actix`
5. 使用数据库：`PostgreSQL`
6. 链接数据库的库：`SQLx`

# 1 开胃菜
## 1.1 自建 `TCP Server`
* 编写 `TCP Server` 与 `Client` 通信

### std::net 模块
* 标准库的 `std::net` 模块，提供网络基本功能
* 支持 `TCP` 和 `UDP` 通信
* 主要使用 `TcpListener` 和 `TcpStream`


- Cargo.toml

```rust
[workspace]

members = ["tcpserver", "tcpclient"]

```

- tcpserver

```rust
use std::net::TcpListener;
use std::io::{Read, Write};

fn main() {
    // 绑定到 3000 端口
    let listener = TcpListener::bind("127.0.0.1:3000").unwrap();
    println!("Running on port 3000...");

    // 只接受一次请求使用 accept，而实际很少用，因为要持续监听进来的连接
    // let result = listener.accept().unwrap();

    /*
        incoming 返回迭代器，迭代器它就会监听 listener 接收到的连接
        而每个连接就代表接收到的字节流，这个字节流的类型就是 TcpStream
        数据就可以在 TcpStream 上传输和接收
        对 TcpStream 的读写是使用原始字节来完成的

        运行：cargo run -p tcpserver
     */
    for stream in listener.incoming() {
        // 使用 unwrap 简单处理下，如果没 err 就返回 stream，否则 panic
        let mut stream = stream.unwrap();
        println!("Connection established!");

        let mut buffer = [0; 1024];
        stream.read(&mut buffer).unwrap();
        stream.write(&mut buffer).unwrap();
    }

}
```

- tcpclient

```rust
use std::net::TcpStream;
use std::io::{Read, Write};
use std::str;


// 运行 client：cargo run -p tcpclient

fn main() {
    let mut stream = TcpStream::connect("127.0.0.1:3000").unwrap();

    // 向服务器传输消息
    // 传输时候需要使用原始字节，所以这里用 as_bytes 这个方法
    stream.write("Hello".as_bytes()).unwrap();
    println!("Hello, world!");

    
    // 从服务器接收消息
    let mut buffer = [0; 5];
    // 读取服务器内容到 buffer
    stream.read(&mut buffer).unwrap();

    // 将 buffer 内容转为 utf-8 字符串
    println!("Response from server:{:?}", str::from_utf8(&buffer).unwrap());
}
```


> 运行：`$ cargo run -p tcpserver` 启动服务端，运行 `$ cargo run -p tcpclient` 启动客户端。
{: .prompt-info }


## 1.2 构建 `HTTP(Web) Server`

![image](/assets/images/rust/web_server/web_server.png)

1. 客户端发起请求到 `Server`
2. `Server（引用了 HTTP Library）`把请求发给路由器
3. 路由器了决定调用哪个 `Handler（也引用了 HTTP Library）`
4. `Handler` 处理 `HTTP` 请求，并返回 `Response` 给客户端


> Rust 中没有内置的 `HTTP` 支持
{: .prompt-info }

### `Web Server` 组成
* `Server`：监听进来的 `TCP` 字节流，会调用 `HTTP Library`
* `HTTP Library`：
1. 可以解释字节流，把它转化为 `HTTP` 请求，然后请求会发送给 `Router`
2. 把 `HTTP` 响应转化回字节流
* `Router`：接受 `HTTP` 请求，并决定调用哪个 `Handler`
* `Handler`：
1. 处理 `HTTP` 请求，构建 `HTTP` 响应，
2. 使用 `HTTP Library` 把 `HTTP` 响应转化回字节流返回给客户端

### 构建步骤
1. 解析 `HTTP` 请求消息
2. 构建 `HTTP` 响应消息
3. 路由和 `Handler`
4. 测试 `Web Server`


![image](/assets/images/rust/web_server/web_server_struct.png)

* 上边三个数据结构都要实现下图三个 `trait`

![image](/assets/images/rust/web_server/web_struct_trait.png)


![image](/assets/images/rust/web_server/web_server_req.png)


#### 1 `HTTP` 请求

* httprequest.rs

```rust
use std::{collections::HashMap, ffi::FromVecWithNulError};


#[derive(Debug, PartialEq)]
pub enum Method {
    Get,
    Post,
    Uninitialized,
}

impl From<&str> for Method {
    fn from(s: &str) -> Method {
        match s {
            "GET" => Method::Get,
            "POST" => Method::Post,
            _ => Method::Uninitialized,
        }
    }
}

#[derive(Debug, PartialEq)]
pub enum Version {
    V1_1,
    V2_0,
    Uninitialized,
}

impl From<&str> for Version {
    fn from(s: &str) -> Version {
        match s {
            "HTTP/1.1" => Version::V1_1,
            _ => Version::Uninitialized,
        }
    }
}

#[derive(Debug, PartialEq)]
pub enum Resource {
    Path(String),
} 

#[derive(Debug)]
pub struct HttpRequest {
    pub method: Method,
    pub version: Version,
    pub resource: Resource,
    pub headers: HashMap<String, String>,
    pub msg_body: String,
}

impl From<String> for HttpRequest {
    fn from(req: String) -> Self {
        // 定义变量设置为初始状态
        let mut parsed_method = Method::Uninitialized;
        let mut parsed_version = Version::V1_1;
        let mut parsed_resource = Resource::Path("".to_string());
        let mut parsed_headers = HashMap::new();
        let mut parsed_msg_body = "";

        for line in req.lines() {
            if line.contains("HTTP") {
                let (method, resource, version) = process_req_line(line);
                parsed_method = method;
                parsed_resource = resource;
                parsed_version = version;
            } else if line.contains(":") {
                let (key, value) = process_header_line(line);
                parsed_headers.insert(key, value); // 将 header 写如 parsed_headers
            } else if line.len() == 0 {
                
            } else {
                parsed_msg_body = line;
            }
        }
        HttpRequest { method: parsed_method, 
            version: parsed_version, 
            resource: parsed_resource, 
            headers: parsed_headers, 
            msg_body: parsed_msg_body.to_string() }
    }


    
}

fn process_req_line(s: &str) -> (Method, Resource, Version) {
    let mut words = s.split_whitespace();
    let method = words.next().unwrap();
    let resource = words.next().unwrap();
    let version = words.next().unwrap();
    (method.into(), Resource::Path(resource.to_string()), version.into())
}

fn process_header_line(s: &str) -> (String, String) {
    let mut header = s.split(":");
    let mut key = String::from("");
    let mut value = String::from("");

    if let Some(k) = header.next() {
        key = k.to_string();
    }
    if let Some(v) = header.next() {
        value = v.to_string();
    }

    (key, value)
}

// 测试代码
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_method_into() {
        /*
            由于 Method 实现了 From<&str>，
            所以这里可使用 into 将字符串转为 Method
         */
        let m: Method = "GET".into();
        assert_eq!(m, Method::Get);
    }

    #[test]
    fn test_version_into() {
        let v: Version = "HTTP/1.1".into();
        assert_eq!(v, Version::V1_1);
    }

    #[test]
    fn test_read_http() {
        let s: String = String::from("GET /greeting HTTP/1.1\r\nHost: localhost:3000\r\nUser-Agent: curl/7.71.1\r\nAccept: */*\r\n\r\n");
        let mut headers_expected = HashMap::new();
        headers_expected.insert("Host".into(), " localhost".into());
        headers_expected.insert("Accept".into(), " */*".into());
        headers_expected.insert("User-Agent".into(), " curl/7.71.1".into());

        let req: HttpRequest = s.into();
        assert_eq!(Method::Get, req.method);
        assert_eq!(Version::V1_1, req.version);
        assert_eq!(Resource::Path("/greeting".to_string()), req.resource);
        assert_eq!(headers_expected, req.headers);


    }
}
```

#### 2 `HTTP` 响应

![image](/assets/images/rust/web_server/web_response.png)

![image](/assets/images/rust/web_server/resp_trait.png)


* httpresponse.rs

```rust
use std::collections::HashMap;
use std::io::{Read, Write, Error};

use crate::httprequest;

// PartialEq 使其成员可与其他值进行比较
#[derive(Debug, PartialEq, Clone)]
pub struct HttpResponse<'a> {
    version: &'a str,
    status_code: &'a str,
    status_text: &'a str,
    headers: Option<HashMap<&'a str, &'a str>>,
    body: Option<String>,
}

impl<'a> Default for HttpResponse<'a> {
    fn default() -> Self {
        Self { 
            version: "HTTP/1.1".into(), 
            status_code: "200".into(), 
            status_text: "OK".into(), 
            headers: None, 
            body: None 
        }
    }   
}

impl<'a> From<HttpResponse<'a>> for String {
    fn from(value: HttpResponse<'a>) -> Self {
        let v1 = value.clone();
        format!(
            "{} {} {}\r\n{}Content-Length: {}\r\n\r\n{}", 
            &v1.version(),
            &v1.status_code(),
            &v1.status_text(),
            &v1.headers(),
            &value.body.unwrap().len(),
            &v1.body()
         )
    }
}

impl<'a> HttpResponse<'a> {
    pub fn new(
        status_code: &'a str, 
        headers: Option<HashMap<&'a str, &'a str>>, 
        body: Option<String>
    ) -> HttpResponse<'a> {
        let mut response: HttpResponse<'a> = HttpResponse::default();

        if status_code != "200" {
            response.status_code = status_code.into();
        };
        response.headers = match &headers {
            Some(_h) => headers,
            None => {
                let mut h = HashMap::new();
                h.insert("Content-Type", "text/html");
                Some(h)
            }
        };

        response.status_text = match response.status_code {
            "200" => "OK".into(),
            "400" => "Bad Request".into(),
            "404" => "404 Not Found".into(),
            "500" => "Internal Server Error".into(),
            _ => "Not Found".into(),
        };

        response.body = body;
         
        response
    }

    // 接收 tcpstream 为参数并且实现了 Write
    pub fn send_response(&self, write_stream:&mut impl Write) -> Result<(), Error> {
        let res = self.clone();
        let response_string: String = String::from(res);
        // 把字符串发送到 tcpstream
        let _ = write!(write_stream, "{}", response_string);

        Ok(())
    }

    fn version(&self) -> &str {
        self.version
    }

    fn status_code(&self) -> &str {
        self.status_code
    }

    fn status_text(&self) -> &str {
        self.status_text
    } 

    fn headers(&self) -> String { 
        let map: HashMap<&str, &str> = self.headers.clone().unwrap();
        let mut header_string: String = "".into();
        for (k, v) in map.iter() {
            header_string = format!("{}{}:{}\r\n", header_string, k, v);
        }
        header_string
    }

    pub fn body(&self) -> &str {
        match &self.body {
            Some(b) => b.as_str(),
            None => "",
        }
    }

}


#[cfg(test)]
mod test {
    use super::*;


    // 测试 new 函数
    #[test]
    fn test_response_struct_creation_200() {
        // 通过 new 函数创建
        let response_actual = HttpResponse::new(
            "200",
            None,
            Some("xxxx".into()),
        );

        // 直接创建
        let response_expected = HttpResponse {
            version: "HTTP/1.1",
            status_code: "200",
            status_text: "OK",
            headers: {
                let mut h = HashMap::new();
                h.insert("Content-Type", "text/html");
                Some(h)
            },
            body: Some("xxxx".into()),
        };
        assert_eq!(response_actual, response_expected);
    }

    #[test]
    fn test_response_struct_creation_404() {
        // 通过 new 函数创建
        let response_actual = HttpResponse::new(
            "404",
            None,
            Some("xxxx".into()),
        );

        // 直接创建
        let response_expected = HttpResponse {
            version: "HTTP/1.1",
            status_code: "404",
            status_text: "404 Not Found",
            headers: {
                let mut h = HashMap::new();
                h.insert("Content-Type", "text/html");
                Some(h)
            },
            body: Some("xxxx".into()),
        };
        assert_eq!(response_actual, response_expected);
    }

    
    #[test]
    fn test_http_response_creation() {
        let response_expected = HttpResponse {
            version: "HTTP/1.1",
            status_code: "404",
            status_text: "404 Not Found",
            headers: {
                let mut h = HashMap::new();
                h.insert("Content-Type", "text/html");
                Some(h)
            },
            body: Some("xxxx".into()),
        };

        let http_string: String = response_expected.into();

        let actual_string = "HTTP/1.1 404 404 Not Found\r\nContent-Type:text/html\r\nContent-Length: 4\r\n\r\nxxxx";

        assert_eq!(http_string, actual_string);

    }

}
```

#### 3 构建 server 模块

* httpserver 里的 Cargo.toml，设置依赖 http 库

```rust
[package]
name = "httpserver"
version = "0.1.0"
edition = "2021"


[dependencies]
# 依赖 workspace 里的 http 库
http = {path = "../http"} 
```


* server.rs


* router.rs

* handler.rs

