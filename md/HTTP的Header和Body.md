HTTP的Header和Body

+ HOST：⽬标主机。**注意：不是在⽹络上⽤于寻址的，⽽是在⽬标服务器上⽤于定位⼦服务器的。**寻址是DNS做的事情。

+ Content-Length：Body的长度。因为涉及二进制的发送，读取Body的时候不能约定结束符号，所以这里客户端发送一个长度，服务器直接读取这个长度就好。

+ Content-Type：指定Body的类型。主要有四类：

  + **1. text/html**

    请求 Web ⻚⾯是返回响应的类型，Body 中返回 html ⽂本。格式如下：

    ![image-20220510103019321](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20220510103019321.png)

  + **x-www-form-urlencoded**

    Web页面纯文本表单的提交方式

    ![image-20220510103841888](HTTP%E7%9A%84Header%E5%92%8CBody.assets/image-20220510103841888.png)

  + **multipart/form-data**

    Web ⻚⾯含有⼆进制⽂件时的提交⽅式。

    ![image-20220510105025010](HTTP%E7%9A%84Header%E5%92%8CBody.assets/image-20220510105025010.png)

  + **application/json , image/jpeg , application/zip ...**

    + application/json

      ![image-20220510110225231](HTTP%E7%9A%84Header%E5%92%8CBody.assets/image-20220510110225231.png)

    +  image/jpeg

      ![image-20220510110334610](HTTP%E7%9A%84Header%E5%92%8CBody.assets/image-20220510110334610.png)

      ```java
      @POST("users/{id}/avatar")
      Call<User> updateAvatar(@Path("id") String id, @Body RequestBody
      avatar);
      
      RequestBody avatarBody =
      RequestBody.create(MediaType.parse("image/jpeg"), avatarFile);
      api.updateAvatar(id, avatarBody)
      ```

+ **Range / Accept-Ranges**

  按范围取数据

  Accept-Ranges: bytes 响应报⽂中出现，表示服务器⽀持按字节来取范围数据

  Range: bytes=start-end 请求报⽂中出现，表示要取哪段数据

  Content-Range:start-end/total 响应报⽂中出现，表示发送的是哪段数据

  作⽤：断点续传、多线程下载