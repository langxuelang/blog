#volley源码学习

>之前一直对于源码学习抱着一种又爱又恨的心情。爱的是因为知道源码有一些特别好的设计思路，可以让自己借鉴，而且对于设计模式来说是最好的实战场。那为啥还会恨呢，曾经很多次下载了很多开源库的源码，可是看的看的就感觉云里雾里，不知所踪。心中没有一个总体的框架，总感觉看的细如牛毛，一叶障目。今天又找时间翻出最简单的volley，准备从头再看一遍。没想到收获很多，写下这篇文章，用来记录。

##volley是什么
volley是一个封装好的网络库，是把httpclient或HttpURLConnection又封装了一层，加上了线程，队列，缓存等机制，让网络请求更加容易。

volley是google官方推出的一个开源项目，专门适用于android轻量的请求。但是对于大文件等支持不好。

volley整个源码都是用了接口编程的思想，这也比较符合设计模式的优秀实践。

所以看volley源码会对于接口编程有较深入的理解。

除了volley之外，还有一个retrofit网络框架，是对okhttp的封装,这个框架现在特别火，而且如果你想领略设计模式之美的话，这个retrofit源码必须看，等之后再写一篇分析retrofit的文章

##volley工作流程

首先先放一张google官方给的流程图，虽然比较简单，但是可以在心中有一个大致的概念

![](1.png)

首先我们从volley调用入手，找到调用入口，这样就可以按图索骥，一点点摸索出整个的流程。

```
RequestQueue queue = Volley.newRequestQueue(this);
String url ="https://www.gaotenglife.com";

// Request a string response from the provided URL.
StringRequest stringRequest = new StringRequest(Request.Method.GET, url,
            new Response.Listener<String>() {
    @Override
    public void onResponse(String response) {
        // Display the first 500 characters of the response string.
        mTextView.setText("Response is: "+ response.substring(0,500));
    }
}, new Response.ErrorListener() {
    @Override
    public void onErrorResponse(VolleyError error) {
        mTextView.setText("That didn't work!");
    }
});
// Add the request to the RequestQueue.
queue.add(stringRequest);

```



