<audio title="34 _ 自己动手写高性能HTTP服务器（三）：TCP字节流处理和HTTP协议实现" src="https://static001.geekbang.org/resource/audio/f6/fb/f6ee76bc2b5a07fce8b463339e5a27fb.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第34讲，欢迎回来。</p><p>这一讲，我们延续第33讲的话题，继续解析高性能网络编程框架的字节流处理部分，并为网络编程框架增加HTTP相关的功能，在此基础上完成HTTP高性能服务器的编写。</p><h2>buffer对象</h2><p>你肯定在各种语言、各种框架里面看到过不同的buffer对象，buffer，顾名思义，就是一个缓冲区对象，缓存了从套接字接收来的数据以及需要发往套接字的数据。</p><p>如果是从套接字接收来的数据，事件处理回调函数在不断地往buffer对象增加数据，同时，应用程序需要不断把buffer对象中的数据处理掉，这样，buffer对象才可以空出新的位置容纳更多的数据。</p><p>如果是发往套接字的数据，应用程序不断地往buffer对象增加数据，同时，事件处理回调函数不断调用套接字上的发送函数将数据发送出去，减少buffer对象中的写入数据。</p><p>可见，buffer对象是同时可以作为输入缓冲（input buffer）和输出缓冲（output buffer）两个方向使用的，只不过，在两种情形下，写入和读出的对象是有区别的。</p><p>这张图描述了buffer对象的设计。</p><p><img src="https://static001.geekbang.org/resource/image/44/bb/44eaf37e860212a5c6c9e7f8dc2560bb.png?wh=946*316" alt=""><br>
下面是buffer对象的数据结构。</p><pre><code>//数据缓冲区
struct buffer {
    char *data;          //实际缓冲
    int readIndex;       //缓冲读取位置
    int writeIndex;      //缓冲写入位置
    int total_size;      //总大小
};
</code></pre><!-- [[[read_end]]] --><p>buffer对象中的writeIndex标识了当前可以写入的位置；readIndex标识了当前可以读出的数据位置，图中红色部分从readIndex到writeIndex的区域是需要读出数据的部分，而绿色部分从writeIndex到缓存的最尾端则是可以写出的部分。</p><p>随着时间的推移，当readIndex和writeIndex越来越靠近缓冲的尾端时，前面部分的front_space_size区域变得会很大，而这个区域的数据已经是旧数据，在这个时候，就需要调整一下整个buffer对象的结构，把红色部分往左侧移动，与此同时，绿色部分也会往左侧移动，整个缓冲区的可写部分就会变多了。</p><p>make_room函数就是起这个作用的，如果右边绿色的连续空间不足以容纳新的数据，而最左边灰色部分加上右边绿色部分一起可以容纳下新数据，就会触发这样的移动拷贝，最终红色部分占据了最左边，绿色部分占据了右边，右边绿色的部分成为一个连续的可写入空间，就可以容纳下新的数据。下面的一张图解释了这个过程。</p><p><img src="https://static001.geekbang.org/resource/image/63/80/638e76a9f926065a72de9116192ef780.png?wh=1046*622" alt=""><br>
下面是make_room的具体实现。</p><pre><code>void make_room(struct buffer *buffer, int size) {
    if (buffer_writeable_size(buffer) &gt;= size) {
        return;
    }
    //如果front_spare和writeable的大小加起来可以容纳数据，则把可读数据往前面拷贝
    if (buffer_front_spare_size(buffer) + buffer_writeable_size(buffer) &gt;= size) {
        int readable = buffer_readable_size(buffer);
        int i;
        for (i = 0; i &lt; readable; i++) {
            memcpy(buffer-&gt;data + i, buffer-&gt;data + buffer-&gt;readIndex + i, 1);
        }
        buffer-&gt;readIndex = 0;
        buffer-&gt;writeIndex = readable;
    } else {
        //扩大缓冲区
        void *tmp = realloc(buffer-&gt;data, buffer-&gt;total_size + size);
        if (tmp == NULL) {
            return;
        }
        buffer-&gt;data = tmp;
        buffer-&gt;total_size += size;
    }
}
</code></pre><p>当然，如果红色部分占据过大，可写部分不够，会触发缓冲区的扩大操作。这里我通过调用realloc函数来完成缓冲区的扩容。</p><p>下面这张图对此做了解释。</p><p><img src="https://static001.geekbang.org/resource/image/9f/ba/9f66d628572b0ef5b7d9d5989c7a14ba.png?wh=1248*500" alt=""></p><h2>套接字接收数据处理</h2><p>套接字接收数据是在tcp_connection.c中的handle_read来完成的。在这个函数里，通过调用buffer_socket_read函数接收来自套接字的数据流，并将其缓冲到buffer对象中。之后你可以看到，我们将buffer对象和tcp_connection对象传递给应用程序真正的处理函数messageCallBack来进行报文的解析工作。这部分的样例在HTTP报文解析中会展开。</p><pre><code>int handle_read(void *data) {
    struct tcp_connection *tcpConnection = (struct tcp_connection *) data;
    struct buffer *input_buffer = tcpConnection-&gt;input_buffer;
    struct channel *channel = tcpConnection-&gt;channel;

    if (buffer_socket_read(input_buffer, channel-&gt;fd) &gt; 0) {
        //应用程序真正读取Buffer里的数据
        if (tcpConnection-&gt;messageCallBack != NULL) {
            tcpConnection-&gt;messageCallBack(input_buffer, tcpConnection);
        }
    } else {
        handle_connection_closed(tcpConnection);
    }
}
</code></pre><p>在buffer_socket_read函数里，调用readv往两个缓冲区写入数据，一个是buffer对象，另外一个是这里的additional_buffer，之所以这样做，是担心buffer对象没办法容纳下来自套接字的数据流，而且也没有办法触发buffer对象的扩容操作。通过使用额外的缓冲，一旦判断出从套接字读取的数据超过了buffer对象里的实际最大可写大小，就可以触发buffer对象的扩容操作，这里buffer_append函数会调用前面介绍的make_room函数，完成buffer对象的扩容。</p><pre><code>int buffer_socket_read(struct buffer *buffer, int fd) {
    char additional_buffer[INIT_BUFFER_SIZE];
    struct iovec vec[2];
    int max_writable = buffer_writeable_size(buffer);
    vec[0].iov_base = buffer-&gt;data + buffer-&gt;writeIndex;
    vec[0].iov_len = max_writable;
    vec[1].iov_base = additional_buffer;
    vec[1].iov_len = sizeof(additional_buffer);
    int result = readv(fd, vec, 2);
    if (result &lt; 0) {
        return -1;
    } else if (result &lt;= max_writable) {
        buffer-&gt;writeIndex += result;
    } else {
        buffer-&gt;writeIndex = buffer-&gt;total_size;
        buffer_append(buffer, additional_buffer, result - max_writable);
    }
    return result;
}
</code></pre><h2>套接字发送数据处理</h2><p>当应用程序需要往套接字发送数据时，即完成了read-decode-compute-encode过程后，通过往buffer对象里写入encode以后的数据，调用tcp_connection_send_buffer，将buffer里的数据通过套接字缓冲区发送出去。</p><pre><code>int tcp_connection_send_buffer(struct tcp_connection *tcpConnection, struct buffer *buffer) {
    int size = buffer_readable_size(buffer);
    int result = tcp_connection_send_data(tcpConnection, buffer-&gt;data + buffer-&gt;readIndex, size);
    buffer-&gt;readIndex += size;
    return result;
}
</code></pre><p>如果发现当前channel没有注册WRITE事件，并且当前tcp_connection对应的发送缓冲无数据需要发送，就直接调用write函数将数据发送出去。如果这一次发送不完，就将剩余需要发送的数据拷贝到当前tcp_connection对应的发送缓冲区中，并向event_loop注册WRITE事件。这样数据就由框架接管，应用程序释放这部分数据。</p><pre><code>//应用层调用入口
int tcp_connection_send_data(struct tcp_connection *tcpConnection, void *data, int size) {
    size_t nwrited = 0;
    size_t nleft = size;
    int fault = 0;

    struct channel *channel = tcpConnection-&gt;channel;
    struct buffer *output_buffer = tcpConnection-&gt;output_buffer;

    //先往套接字尝试发送数据
    if (!channel_write_event_registered(channel) &amp;&amp; buffer_readable_size(output_buffer) == 0) {
        nwrited = write(channel-&gt;fd, data, size);
        if (nwrited &gt;= 0) {
            nleft = nleft - nwrited;
        } else {
            nwrited = 0;
            if (errno != EWOULDBLOCK) {
                if (errno == EPIPE || errno == ECONNRESET) {
                    fault = 1;
                }
            }
        }
    }

    if (!fault &amp;&amp; nleft &gt; 0) {
        //拷贝到Buffer中，Buffer的数据由框架接管
        buffer_append(output_buffer, data + nwrited, nleft);
        if (!channel_write_event_registered(channel)) {
            channel_write_event_add(channel);
        }
    }

    return nwrited;
}
</code></pre><h2>HTTP协议实现</h2><p>下面，我们在TCP的基础上，加入HTTP的功能。</p><p>为此，我们首先定义了一个http_server结构，这个http_server本质上就是一个TCPServer，只不过暴露给应用程序的回调函数更为简单，只需要看到http_request和http_response结构。</p><pre><code>typedef int (*request_callback)(struct http_request *httpRequest, struct http_response *httpResponse);

struct http_server {
    struct TCPserver *tcpServer;
    request_callback requestCallback;
};
</code></pre><p>在http_server里面，重点是需要完成报文的解析，将解析的报文转化为http_request对象，这件事情是通过http_onMessage回调函数来完成的。在http_onMessage函数里，调用的是parse_http_request完成报文解析。</p><pre><code>// buffer是框架构建好的，并且已经收到部分数据的情况下
// 注意这里可能没有收到全部数据，所以要处理数据不够的情形
int http_onMessage(struct buffer *input, struct tcp_connection *tcpConnection) {
    yolanda_msgx(&quot;get message from tcp connection %s&quot;, tcpConnection-&gt;name);

    struct http_request *httpRequest = (struct http_request *) tcpConnection-&gt;request;
    struct http_server *httpServer = (struct http_server *) tcpConnection-&gt;data;

    if (parse_http_request(input, httpRequest) == 0) {
        char *error_response = &quot;HTTP/1.1 400 Bad Request\r\n\r\n&quot;;
        tcp_connection_send_data(tcpConnection, error_response, sizeof(error_response));
        tcp_connection_shutdown(tcpConnection);
    }

    //处理完了所有的request数据，接下来进行编码和发送
    if (http_request_current_state(httpRequest) == REQUEST_DONE) {
        struct http_response *httpResponse = http_response_new();

        //httpServer暴露的requestCallback回调
        if (httpServer-&gt;requestCallback != NULL) {
            httpServer-&gt;requestCallback(httpRequest, httpResponse);
        }

        //将httpResponse发送到套接字发送缓冲区中
        struct buffer *buffer = buffer_new();
        http_response_encode_buffer(httpResponse, buffer);
        tcp_connection_send_buffer(tcpConnection, buffer);

        if (http_request_close_connection(httpRequest)) {
            tcp_connection_shutdown(tcpConnection);
            http_request_reset(httpRequest);
        }
    }
}
</code></pre><p>还记得<a href="https://time.geekbang.org/column/article/132443">第16讲中</a>讲到的HTTP协议吗？我们从16讲得知，HTTP通过设置回车符、换行符作为HTTP报文协议的边界。</p><p><img src="https://static001.geekbang.org/resource/image/6d/5a/6d91c7c2a0224f5d4bad32a0f488765a.png?wh=942*324" alt=""><br>
parse_http_request的思路就是寻找报文的边界，同时记录下当前解析工作所处的状态。根据解析工作的前后顺序，把报文解析的工作分成REQUEST_STATUS、REQUEST_HEADERS、REQUEST_BODY和REQUEST_DONE四个阶段，每个阶段解析的方法各有不同。</p><p>在解析状态行时，先通过定位CRLF回车换行符的位置来圈定状态行，进入状态行解析时，再次通过查找空格字符来作为分隔边界。</p><p>在解析头部设置时，也是先通过定位CRLF回车换行符的位置来圈定一组key-value对，再通过查找冒号字符来作为分隔边界。</p><p>最后，如果没有找到冒号字符，说明解析头部的工作完成。</p><p>parse_http_request函数完成了HTTP报文解析的四个阶段:</p><pre><code>int parse_http_request(struct buffer *input, struct http_request *httpRequest) {
    int ok = 1;
    while (httpRequest-&gt;current_state != REQUEST_DONE) {
        if (httpRequest-&gt;current_state == REQUEST_STATUS) {
            char *crlf = buffer_find_CRLF(input);
            if (crlf) {
                int request_line_size = process_status_line(input-&gt;data + input-&gt;readIndex, crlf, httpRequest);
                if (request_line_size) {
                    input-&gt;readIndex += request_line_size;  // request line size
                    input-&gt;readIndex += 2;  //CRLF size
                    httpRequest-&gt;current_state = REQUEST_HEADERS;
                }
            }
        } else if (httpRequest-&gt;current_state == REQUEST_HEADERS) {
            char *crlf = buffer_find_CRLF(input);
            if (crlf) {
                /**
                 *    &lt;start&gt;-------&lt;colon&gt;:-------&lt;crlf&gt;
                 */
                char *start = input-&gt;data + input-&gt;readIndex;
                int request_line_size = crlf - start;
                char *colon = memmem(start, request_line_size, &quot;: &quot;, 2);
                if (colon != NULL) {
                    char *key = malloc(colon - start + 1);
                    strncpy(key, start, colon - start);
                    key[colon - start] = '\0';
                    char *value = malloc(crlf - colon - 2 + 1);
                    strncpy(value, colon + 1, crlf - colon - 2);
                    value[crlf - colon - 2] = '\0';

                    http_request_add_header(httpRequest, key, value);

                    input-&gt;readIndex += request_line_size;  //request line size
                    input-&gt;readIndex += 2;  //CRLF size
                } else {
                    //读到这里说明:没找到，就说明这个是最后一行
                    input-&gt;readIndex += 2;  //CRLF size
                    httpRequest-&gt;current_state = REQUEST_DONE;
                }
            }
        }
    }
    return ok;
}
</code></pre><p>处理完了所有的request数据，接下来进行编码和发送的工作。为此，创建了一个http_response对象，并调用了应用程序提供的编码函数requestCallback，接下来，创建了一个buffer对象，函数http_response_encode_buffer用来将http_response中的数据，根据HTTP协议转换为对应的字节流。</p><p>可以看到，http_response_encode_buffer设置了如Content-Length等http_response头部，以及http_response的body部分数据。</p><pre><code>void http_response_encode_buffer(struct http_response *httpResponse, struct buffer *output) {
    char buf[32];
    snprintf(buf, sizeof buf, &quot;HTTP/1.1 %d &quot;, httpResponse-&gt;statusCode);
    buffer_append_string(output, buf);
    buffer_append_string(output, httpResponse-&gt;statusMessage);
    buffer_append_string(output, &quot;\r\n&quot;);

    if (httpResponse-&gt;keep_connected) {
        buffer_append_string(output, &quot;Connection: close\r\n&quot;);
    } else {
        snprintf(buf, sizeof buf, &quot;Content-Length: %zd\r\n&quot;, strlen(httpResponse-&gt;body));
        buffer_append_string(output, buf);
        buffer_append_string(output, &quot;Connection: Keep-Alive\r\n&quot;);
    }

    if (httpResponse-&gt;response_headers != NULL &amp;&amp; httpResponse-&gt;response_headers_number &gt; 0) {
        for (int i = 0; i &lt; httpResponse-&gt;response_headers_number; i++) {
            buffer_append_string(output, httpResponse-&gt;response_headers[i].key);
            buffer_append_string(output, &quot;: &quot;);
            buffer_append_string(output, httpResponse-&gt;response_headers[i].value);
            buffer_append_string(output, &quot;\r\n&quot;);
        }
    }

    buffer_append_string(output, &quot;\r\n&quot;);
    buffer_append_string(output, httpResponse-&gt;body);
}
</code></pre><h2>完整的HTTP服务器例子</h2><p>现在，编写一个HTTP服务器例子就变得非常简单。</p><p>在这个例子中，最主要的部分是onRequest callback函数，这里，onRequest方法已经在parse_http_request之后，可以根据不同的http_request的信息，进行计算和处理。例子程序里的逻辑非常简单，根据http request的URL path，返回了不同的http_response类型。比如，当请求为根目录时，返回的是200和HTML格式。</p><pre><code>#include &lt;lib/acceptor.h&gt;
#include &lt;lib/http_server.h&gt;
#include &quot;lib/common.h&quot;
#include &quot;lib/event_loop.h&quot;

//数据读到buffer之后的callback
int onRequest(struct http_request *httpRequest, struct http_response *httpResponse) {
    char *url = httpRequest-&gt;url;
    char *question = memmem(url, strlen(url), &quot;?&quot;, 1);
    char *path = NULL;
    if (question != NULL) {
        path = malloc(question - url);
        strncpy(path, url, question - url);
    } else {
        path = malloc(strlen(url));
        strncpy(path, url, strlen(url));
    }

    if (strcmp(path, &quot;/&quot;) == 0) {
        httpResponse-&gt;statusCode = OK;
        httpResponse-&gt;statusMessage = &quot;OK&quot;;
        httpResponse-&gt;contentType = &quot;text/html&quot;;
        httpResponse-&gt;body = &quot;&lt;html&gt;&lt;head&gt;&lt;title&gt;This is network programming&lt;/title&gt;&lt;/head&gt;&lt;body&gt;&lt;h1&gt;Hello, network programming&lt;/h1&gt;&lt;/body&gt;&lt;/html&gt;&quot;;
    } else if (strcmp(path, &quot;/network&quot;) == 0) {
        httpResponse-&gt;statusCode = OK;
        httpResponse-&gt;statusMessage = &quot;OK&quot;;
        httpResponse-&gt;contentType = &quot;text/plain&quot;;
        httpResponse-&gt;body = &quot;hello, network programming&quot;;
    } else {
        httpResponse-&gt;statusCode = NotFound;
        httpResponse-&gt;statusMessage = &quot;Not Found&quot;;
        httpResponse-&gt;keep_connected = 1;
    }

    return 0;
}


int main(int c, char **v) {
    //主线程event_loop
    struct event_loop *eventLoop = event_loop_init();

    //初始tcp_server，可以指定线程数目，如果线程是0，就是在这个线程里acceptor+i/o；如果是1，有一个I/O线程
    //tcp_server自己带一个event_loop
    struct http_server *httpServer = http_server_new(eventLoop, SERV_PORT, onRequest, 2);
    http_server_start(httpServer);

    // main thread for acceptor
    event_loop_run(eventLoop);
}
</code></pre><p>运行这个程序之后，我们可以通过浏览器和curl命令来访问它。你可以同时开启多个浏览器和curl命令，这也证明了我们的程序是可以满足高并发需求的。</p><pre><code>$curl -v http://127.0.0.1:43211/
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 43211 (#0)
&gt; GET / HTTP/1.1
&gt; Host: 127.0.0.1:43211
&gt; User-Agent: curl/7.54.0
&gt; Accept: */*
&gt;
&lt; HTTP/1.1 200 OK
&lt; Content-Length: 116
&lt; Connection: Keep-Alive
&lt;
* Connection #0 to host 127.0.0.1 left intact
&lt;html&gt;&lt;head&gt;&lt;title&gt;This is network programming&lt;/title&gt;&lt;/head&gt;&lt;body&gt;&lt;h1&gt;Hello, network programming&lt;/h1&gt;&lt;/body&gt;&lt;/html&gt;%
</code></pre><p><img src="https://static001.geekbang.org/resource/image/71/a5/719804f279f057a9a12b5904a39e06a5.png?wh=1106*330" alt=""></p><h2>总结</h2><p>这一讲我们主要讲述了整个编程框架的字节流处理能力，引入了buffer对象，并在此基础上通过增加HTTP的特性，包括http_server、http_request、http_response，完成了HTTP高性能服务器的编写。实例程序利用框架提供的能力，编写了一个简单的HTTP服务器程序。</p><h2>思考题</h2><p>和往常一样，给你布置两道思考题：</p><p>第一道， 你可以试着在HTTP服务器中增加MIME的处理能力，当用户请求/photo路径时，返回一张图片。</p><p>第二道，在我们的开发中，已经有很多面向对象的设计，你可以仔细研读代码，说说你对这部分的理解。</p><p>欢迎你在评论区写下你的思考，也欢迎把这篇文章分享给你的朋友或者同事，一起交流一下。</p>
<style>
    ul {
      list-style: none;
      display: block;
      list-style-type: disc;
      margin-block-start: 1em;
      margin-block-end: 1em;
      margin-inline-start: 0px;
      margin-inline-end: 0px;
      padding-inline-start: 40px;
    }
    li {
      display: list-item;
      text-align: -webkit-match-parent;
    }
    ._2sjJGcOH_0 {
      list-style-position: inside;
      width: 100%;
      display: -webkit-box;
      display: -ms-flexbox;
      display: flex;
      -webkit-box-orient: horizontal;
      -webkit-box-direction: normal;
      -ms-flex-direction: row;
      flex-direction: row;
      margin-top: 26px;
      border-bottom: 1px solid rgba(233,233,233,0.6);
    }
    ._2sjJGcOH_0 ._3FLYR4bF_0 {
      width: 34px;
      height: 34px;
      -ms-flex-negative: 0;
      flex-shrink: 0;
      border-radius: 50%;
    }
    ._2sjJGcOH_0 ._36ChpWj4_0 {
      margin-left: 0.5rem;
      -webkit-box-flex: 1;
      -ms-flex-positive: 1;
      flex-grow: 1;
      padding-bottom: 20px;
    }
    ._2sjJGcOH_0 ._36ChpWj4_0 ._2zFoi7sd_0 {
      font-size: 16px;
      color: #3d464d;
      font-weight: 500;
      -webkit-font-smoothing: antialiased;
      line-height: 34px;
    }
    ._2sjJGcOH_0 ._36ChpWj4_0 ._2_QraFYR_0 {
      margin-top: 12px;
      color: #505050;
      -webkit-font-smoothing: antialiased;
      font-size: 14px;
      font-weight: 400;
      white-space: normal;
      word-break: break-all;
      line-height: 24px;
    }
    ._2sjJGcOH_0 ._10o3OAxT_0 {
      margin-top: 18px;
      border-radius: 4px;
      background-color: #f6f7fb;
    }
    ._2sjJGcOH_0 ._3klNVc4Z_0 {
      display: -webkit-box;
      display: -ms-flexbox;
      display: flex;
      -webkit-box-orient: horizontal;
      -webkit-box-direction: normal;
      -ms-flex-direction: row;
      flex-direction: row;
      -webkit-box-pack: justify;
      -ms-flex-pack: justify;
      justify-content: space-between;
      -webkit-box-align: center;
      -ms-flex-align: center;
      align-items: center;
      margin-top: 15px;
    }
    ._2sjJGcOH_0 ._10o3OAxT_0 ._3KxQPN3V_0 {
      color: #505050;
      -webkit-font-smoothing: antialiased;
      font-size: 14px;
      font-weight: 400;
      white-space: normal;
      word-break: break-word;
      padding: 20px 20px 20px 24px;
    }
    ._2sjJGcOH_0 ._3klNVc4Z_0 {
      display: -webkit-box;
      display: -ms-flexbox;
      display: flex;
      -webkit-box-orient: horizontal;
      -webkit-box-direction: normal;
      -ms-flex-direction: row;
      flex-direction: row;
      -webkit-box-pack: justify;
      -ms-flex-pack: justify;
      justify-content: space-between;
      -webkit-box-align: center;
      -ms-flex-align: center;
      align-items: center;
      margin-top: 15px;
    }
    ._2sjJGcOH_0 ._3Hkula0k_0 {
      color: #b2b2b2;
      font-size: 14px;
    }
</style><ul><li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b3/17/19ea024f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>chs</span>
  </div>
  <div class="_2_QraFYR_0">老师不明白缓冲区为什么要这样设计。用两块内存当做缓冲区，一个用于接收数据，另一个用于发送数据。这两种方式的优缺点能说一下吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里你的理解有点问题，确实是两个buffer对象，一个input_buffer用来接收数据，这个input_buffer对象的写入是框架的handle_read函数来完成的，同时应用程序不端的将input_buffer里的数据取走，这样handle_read就可以不断的将接收缓冲区的数据写入input_buffer。<br><br>另一个buffer对象是output_buffer，应用程序不断的往这个缓冲区里写入待发送的数据，框架里的handle_write函数不端的将缓冲区的数据送到套接字发送缓冲区中。<br><br>缓冲区的设计中，肯定是有一个往缓冲区里写入的，另一个从缓冲区里读取数据，否则就没有缓冲区了，而是临时创建一个个的字节流对象。<br><br>使用缓冲区可以大大减少对内存的消耗。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-15 14:02:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/73/9b/67a38926.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>keepgoing</span>
  </div>
  <div class="_2_QraFYR_0">老师，在tcp_connection.c文件tcp_connection_new方法创建channel时传入的data是tcp_connection类型，但在channel.c中channel_write_event_enable方法会直接从channel-&gt;data中取一个event_loop类型指针出来，阅读了整个tcp框架看起来没有找到直接传入event_loop类型的地方，这里是一个代码bug吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你读的很仔细，我看了一下，确实是有问题的。<br><br>最简单的方法是在channel里面保持一个event_loop对象指针，在构建channel时传过来，在channel.c的channel_write_event_enable方法里直接使用这个对象就可以了。<br><br>如果可以，欢迎你提一个patch过来，感谢~<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-07 20:49:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/d3/80/dd0b26cb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗兆峰</span>
  </div>
  <div class="_2_QraFYR_0">第二题： 用户申请图片的时候可以申请一个GET 方法的request, GET URI version, URI 是图片相对服务器程序的地址，在服务器端程序使用io 函数read&#47;或者mmap 读取图片文件的内容， 并且写到connectedfd 中即可， http response 中的文件类型标记为image&#47;png。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-18 16:34:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/4d/82/2bb78658.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小家伙54</span>
  </div>
  <div class="_2_QraFYR_0">老师，ubuntu20.4运行lib程序会出现段错误，这是怎么回事啊？<br><br>nuc@nuc-NUC8i5BEHS:~&#47;learn&#47;GeekTime&#47;net_prog&#47;yolanda&#47;build&#47;bin$ .&#47;http_server01<br>[msg] set epoll as dispatcher, main thread<br>[msg] add channel fd == 5, main thread<br>[msg] set epoll as dispatcher, Thread-1<br>[msg] add channel fd == 9, Thread-1<br>[msg] event loop thread init and signal, Thread-1<br>[msg] event loop run, Thread-1<br>[msg] event loop thread started, Thread-1<br>[msg] set epoll as dispatcher, Thread-2<br>[msg] add channel fd == 12, Thread-2<br>[msg] event loop thread init and signal, Thread-2<br>[msg] event loop run, Thread-2<br>[msg] event loop thread started, Thread-2<br>[msg] add channel fd == 6, main thread<br>[msg] event loop run, main thread<br>[msg] epoll_wait wakeup, main thread<br>[msg] get message channel fd==6 for read, main thread<br>[msg] activate channel fd == 6, revents=2, main thread<br>[msg] new connection established, socket == 13<br>[msg] connection completed<br>[msg] epoll_wait wakeup, Thread-1<br>[msg] get message channel fd==9 for read, Thread-1<br>[msg] activate channel fd == 9, revents=2, Thread-1<br>[msg] wakeup, Thread-1<br>[msg] add channel fd == 13, Thread-1<br>[msg] epoll_wait wakeup, Thread-1<br>[msg] get message channel fd==13 for read, Thread-1<br>[msg] activate channel fd == 13, revents=2, Thread-1<br>[msg] get message from tcp connection connection-13<br>段错误 (核心已转储)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 貌似是内存访问出错了，你可以看下dump文件，另外，我不能确定是和ubuntu20.4有关。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-09 16:23:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/9d/4a/09a5041e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>TinyCalf</span>
  </div>
  <div class="_2_QraFYR_0">&#47;&#47;初始化一个request对象<br>struct http_request *http_request_new() {<br>    struct http_request *httpRequest = malloc(sizeof(struct http_request));<br>    httpRequest-&gt;method = NULL;<br>    httpRequest-&gt;current_state = REQUEST_STATUS;<br>    httpRequest-&gt;version = NULL;<br>    httpRequest-&gt;url = NULL;<br>    httpRequest-&gt;request_headers = malloc(sizeof(struct http_request) * INIT_REQUEST_HEADER_SIZE);<br>    httpRequest-&gt;request_headers_number = 0;<br>    return httpRequest;<br>}<br>这里的<br>httpRequest-&gt;request_headers = malloc(sizeof(struct http_request) * INIT_REQUEST_HEADER_SIZE);<br>是不是写错了 ；）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有哦，这个其实是一个request_header的数组，直接用指针来表示了，这个数组的最大长度是INIT_REQUEST_HEADER_SIZE。因为http request header就是一个数组。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-17 11:47:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/f6/e3/e4bcd69e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沉淀的梦想</span>
  </div>
  <div class="_2_QraFYR_0">在ubuntu系统上一运行老师的程序就会出现“interrupted by signal 11: SIGSEGV”错误</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我也是ubuntu系统啊，有同学碰到同样的问题么？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-30 12:19:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/df/1e/cea897e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>传说中的成大大</span>
  </div>
  <div class="_2_QraFYR_0">第二个问题 才是把我考到了 我感觉现在我对设计模式的理解并不深,但是我现在感受特深的一点就是单一职责原理 buffer类才套接字的处理 tcpconnect应用层面的处理,而且最近在工作中我也是尝试着画流程图 把每个功能进行细分 分到一个流程分支里面只处理一个逻辑</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-25 10:37:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/4b/11/d7e08b5b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dll</span>
  </div>
  <div class="_2_QraFYR_0">好不容易看完了 打卡纪念一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: 真棒！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-04 02:18:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/48/19/14dd81d9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>铲铲队</span>
  </div>
  <div class="_2_QraFYR_0">make_room 函数就是起这个作用的，如果右边绿色的连续空间不足以容纳新的数据，而最左边灰色部分加上右边绿色部分一起可以容纳下新数据，就会触发这样的移动拷贝，最终红色部分占据了最左边，绿色部分占据了右边，右边绿色的部分成为一个连续的可写入空间，就可以容纳下新的数据<br>----》个人觉得好像不用移动拷贝，数据一部分拷贝满writeable_size,剩余部分拷贝到front_spare_size。即循环缓冲，这样效率更高吧<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这样你不好管理啊，统一部分数据你硬生生把它劈成两份，这两份数据你怎么读呢？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-13 21:14:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/e0/5d/c2867e36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>肥磊</span>
  </div>
  <div class="_2_QraFYR_0">老师，用wenbench测试出现段错误，是什么原因，<br>[msg] get message channel i==0, fd==7, Thread-1<br>[msg] activate channel fd == 7, revents=2, Thread-1<br>[msg] wakeup, Thread-1<br>[msg] add channel fd == 14, Thread-1<br>[msg] poll added channel fd==14, Thread-1<br>[msg] get message channel i==2, fd==14, Thread-1<br>[msg] activate channel fd == 14, revents=2, Thread-1<br>[msg] get message from tcp connection connection-14<br>[1]    2424 segmentation fault (core dumped)  .&#47;http_server01<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我怀疑我哪里有内存拷贝没考虑细致，只能慢慢troubleshooting了，欢迎你debug下，提MR帮我改进哈。&lt;抱拳&gt;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-07 17:52:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/31/17/ab2c27a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>菜鸡互啄</span>
  </div>
  <div class="_2_QraFYR_0">哦 我知道了。front_spare_size是被读了一段之后产生的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好像你悟了 ：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-27 00:03:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/31/17/ab2c27a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>菜鸡互啄</span>
  </div>
  <div class="_2_QraFYR_0">老师你好 为什么要设计front_spare_size？或者说为什么存在front_spare_size？readIndex和writeIndex一开始不是从0开始的吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好像你悟了 ：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-27 00:00:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/a4/9d/95900f70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>T------T</span>
  </div>
  <div class="_2_QraFYR_0">老师好，发现一个memmem函数运行错误的Bug.<br>环境：Ubuntu18.04 GCC 10.3 glic 2.33<br>问题：返回void* 的memmem函数未声明，系统默认调用了返回int的memmem函数。返回值由int强转成char*,导致后续处理出现错误。<br>解决办法：在#include&lt;string.h&gt; 之前添加#define _GNU_SOURCE解决<br>参考：https:&#47;&#47;insidelinuxdev.net&#47;article&#47;a09522.html</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，严格来说，所有和OS环境有关联的函数调用，都需要抽象屏蔽一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-08 20:50:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/cabLXAUXiavXnEckAgo971o4l1CxP4L9wOV2eUGTyKBUicTib6gJyKV9iatM4GG1scz5Ym17GOzXWQEGzhE31tXUtQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>日就月将</span>
  </div>
  <div class="_2_QraFYR_0">老师  http服务器request初始化的时候 http_header申请内存为什么还要乘128</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是因为http request headers的key-value对，128这个值足够用了。要知道，request headers本身是一个map，所以需要指定map的大小。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-17 15:46:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/b9/7c/afe6f1eb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>vv_test</span>
  </div>
  <div class="_2_QraFYR_0">[msg] set poll as dispatcher, main thread<br>[msg] add channel fd == 4, main thread<br>[msg] poll added channel fd==4, main thread<br>[msg] set poll as dispatcher, Thread-1<br>[msg] add channel fd == 7, Thread-1<br>[msg] poll added channel fd==7, Thread-1<br>[msg] event loop thread init and signal, Thread-1<br>[msg] event loop run, Thread-1<br>[msg] event loop thread started, Thread-1<br>poll failed : Invalid argument (22)<br>[msg] set poll as dispatcher, Thread-2<br>[msg] add channel fd == 9, Thread-2<br>[msg] poll added channel fd==9, Thread-2<br>[msg] event loop thread init and signal, Thread-2<br>[msg] event loop run, Thread-2<br>poll failed : Invalid argument (22)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个看上去是poll参数传入出错了，什么系统? </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-03 16:34:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/9c/7d/774e07f9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>study的程序员</span>
  </div>
  <div class="_2_QraFYR_0">缓冲区可以设置为环形的，避免移动</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 管理起来难度挺大</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-15 22:45:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/zkbuproauw8Ov7jjIGYertOQMLGtIzo26bc1m0CsnAHhQ96bpbh4A4jmdE2qm6lccpnr7nnDG93W6JUyDrCjPg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JeQer</span>
  </div>
  <div class="_2_QraFYR_0">没有经过压力测试的服务器怎么能称为高性能呢?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎压测</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-17 11:52:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/8a/f4/a4243808.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>卡布猴纸</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，断连的tcpconnection和channel资源怎么管理的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 直接回收掉。<br><br>int handle_connection_closed(struct tcp_connection *tcpConnection) {<br>    struct event_loop *eventLoop = tcpConnection-&gt;eventLoop;<br>    struct channel *channel = tcpConnection-&gt;channel;<br>    event_loop_remove_channel_event(eventLoop, channel-&gt;fd, channel);<br>    if (tcpConnection-&gt;connectionClosedCallBack != NULL) {<br>        tcpConnection-&gt;connectionClosedCallBack(tcpConnection);<br>    }<br>}</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-01 20:25:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/03/e9/6358059c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GalaxyCreater</span>
  </div>
  <div class="_2_QraFYR_0">在buffer_socket_read函数用了char additional_buffer[INIT_BUFFER_SIZE]这个临时变量，那么预创建buffer来减少内存创建的开销就没效了，最少在读数据上的优化已经没效了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果实际读缓冲区的数据量比较大，后面会通过buffer_append把additional_buffer里面的数据拷贝到buffer中，这样，其实就减少了系统调用的次数，通过空间换取了时间。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-24 19:29:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/98/fa/d87a1432.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>I believe you</span>
  </div>
  <div class="_2_QraFYR_0">老师，用你的程序在linux中使用webbench进行压力测试，每次请求都只有一个成功，其他全都失败，能请问下是什么原因吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: webbench发送的请求是啥？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-11 15:16:46</div>
  </div>
</div>
</div>
</li>
</ul>