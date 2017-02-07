---
title: 使用七牛云存储图片 之 上传图片
date: 2017-01-22 17:35:55
tags: [javascript, qiniu, upload]
---

七牛云是国内算是挺不错的图片存储服务器，免费用户能拥有10G空间，每个月10G的下载流量，对于个人用户用于做个小博客，小网站来说，已经够了。
<!--more--->

七牛云是国内算是挺不错的图片存储服务器，免费用户能拥有10G空间，每个月10G的下载流量，对于个人用户用于做个小博客，小网站来说，已经够了。

但是，七牛的官方开发文档真心会看得一头雾水。

于是，先记录下来这次传图的过程。

在上传图片之前，需要准备两样东西：
1.  `AccessKey`
2.  `SecretKey`

其中， AccessKey 和 SecretKey 能在 个人中心 > 秘钥管理中获得,如图。

1.  创建空间 并 创建配置文件

    选择 **资源主页** ，再选择 **立即添加**，或者选择 **对象存储** ，然后选择 **添加** 。

    然后填上空间的名字，选择区域即可。

    **其中，空间的名字是我们使用代码上传至七牛服务器的第三个参数 `Bucket`**

    可以将这3个常量存储在配置文件`config.qiniu.php`中
    ```
    define('QINIU_SECRET_KEY', 'YOUR_SK');
    define('QINIU_ACCESS_KEY', 'YOUR_AK');
    define('QINIU_DOMAIN', 'YOUR_QINIU_DOMAIN');
    define('QINIU_BUCKET', 'YOUR_BUCKET_NAME');
    ```

2.  获取上传凭证

    上传凭证的作用：
    >客户端上传前需要先从服务端获取上传凭证，并在上传资源时将上传凭证作为请求内容的一部分。不带凭证或带非法凭证的请求将返回 HTTP 错误码 401，代表认证失败。

    生成规则：

    1.  构造上传策略：

        >上传策略是资源上传时附带的一组配置设定。通过这组配置信息，七牛云存储可以了解用户上传的需求：它将上传什么资源，上传到哪个空间，上传结果是回调通知还是使用重定向跳转，是否需要设置反馈信息的内容，以及授权上传的截止时间等等。

        ```
        {

            "scope":               "<Bucket                   string>",
            "deadline":             <UnixTimestamp            uint32>,
            "insertOnly":           <AllowFileUpdating        int>,
            "endUser":             "<EndUserId                string>",
            "returnUrl":           "<RedirectURL              string>",
            "returnBody":          "<ResponseBodyForAppClient string>",
            "callbackUrl":         "<RequestUrlForAppServer   string>",
            "callbackHost":        "<RequestHostForAppServer  string>",
            "callbackBody":        "<RequestBodyForAppServer  string>",
            "callbackBodyType":    "<RequestBodyTypeForAppServer  string>",
            "callbackFetchKey":     <RequestKeyForApp         int>
            "persistentOps":       "<persistentOpsCmds        string>",
            "persistentNotifyUrl": "<persistentNotifyUrl      string>",
            "persistentPipeline":  "<persistentPipeline       string>",
            "saveKey":             "<SaveKey                  string>",
            "fsizeMin":             <FileSizeMin            int64>,
            "fsizeLimit":           <FileSizeLimit            int64>,
            "detectMime":           <AutoDetectMimeType       int>,
            "mimeLimit":           "<MimeLimit                string>"
            "deleteAfterDays":     "<deleteAfterDays          int>"
        }
        ```

        <table>
        <tr><th>字段名称    </th><th>必填 </th><th>说明 </th></tr>
        <tr><td>scope      </td><td>是          </td><td>指定上传的目标资源空间 Bucket 和资源键 Key（key的长度最大为750字节）。有两种格式： ● `<bucket>`，表示允许用户上传文件到指定的 bucket。在这种格式下文件只能新增，若已存在同名资源上传则会失败。  ● `<bucket>`:`<key>`，表示只允许用户上传指定 key 的文件。在这种格式下文件默认允许修改，若已存在同名资源则会被覆盖。如果只希望上传指定 key 的文件，并且不允许修改，那么可以将下面的 `insertOnly` 属性值设为 1</td></tr>
        <tr><td>deadline   </td><td>是          </td><td>上传凭证有效截止时间。`Unix时间戳`，单位为秒。该截止时间为上传完成后，在七牛空间生成文件的校验时间，而非上传的开始时间，一般建议设置为上传开始时间 + 3600s，用户可根据具体的业务场景对凭证截止时间进行调整。</td></tr>
        <tr><td>insertOnly </td><td> 	</td><td>		限定为新增语意。如果设置为非 0 值，则无论 `scope` 设置为什么形式，仅能以新增模式上传文件。</td></tr>
        <tr><td>endUser </td><td> 	</td><td>		唯一属主标识。特殊场景下非常有用，例如根据 App-Client 标识给图片或视频打水印。</td></tr>
        <tr><td>returnUrl </td><td> 	</td><td>		Web 端文件上传成功后，浏览器执行 303 跳转的 URL。通常用于 HTML Form 上传。文件上传成功后会跳转到 `<returnUrl>?upload_ret=<queryString>`，`<queryString>`包含 `returnBody` 内容。如不设置 returnUrl，则直接将 returnBody 的内容返回给客户端。</td></tr>
        <tr><td>returnBody </td><td> 	</td><td>		上传成功后，自定义七牛云最终返回給上传端（在指定 returnUrl 时是携带在跳转路径参数中）的数据。支持魔法变量和自定义变量。returnBody 要求是合法的 JSON 文本。例如 `{"key": $(key), "hash": $(etag), "w": $(imageInfo.width), "h": $(imageInfo.height)}`。</td></tr>
        <tr><td>callbackUrl </td><td> 	</td><td>		上传成功后，七牛云向 App-Server 发送 POST 请求的 URL。必须是公网上可以正常进行 POST 请求并能响应 HTTP/1.1 200 OK 的有效 URL。另外，为了给客户端有一致的体验，我们要求 callbackUrl 返回包 Content-Type 为 "application/json"，即返回的内容必须是合法的 JSON 文本。出于高可用的考虑，本字段允许设置多个 callbackUrl(用英文符号 ; 分隔)，在前一个 callbackUrl 请求失败的时候会依次重试下一个 callbackUrl。一个典型例子是 `http://<ip1>/callback;http://<ip2>/callback`，并同时指定下面的 callbackHost 字段。在 callbackUrl 中使用 ip 的好处是减少了对 dns 解析的依赖，可改善回调的性能和稳定性。</td></tr>
        <tr><td>callbackHost </td><td> 	</td><td>		上传成功后，七牛云向"App-Server"发送回调通知时的 Host 值。 与callbackUrl配合使用，仅当设置了 callbackUrl 时才有效。"callbackHost":"a.example.com"，host域名不加http://</td></tr>
        <tr><td>callbackBody </td><td> 	</td><td>		上传成功后，七牛云向"App-Server"发送Content-Type: application/x-www-form-urlencoded 的POST请求。 该字段"App-Server"可以通过直接读取请求的query来获得，支持魔法变量和自定义变量。callbackBody 要求是合法的 url query string。如：`key=$(key)&hash=$(etag)&w=$(imageInfo.width)&h=$(imageInfo.height)`。</td></tr>
        <tr><td>callbackBodyType </td><td> 	</td><td>		上传成功后，七牛云向"App-Server"发送回调通知`callbackBody`的`Content-Type`。 默认为`application/x-www-form-urlencoded`，也可设置为`application/json`。</td></tr>
        <tr><td>callbackFetchKey </td><td> 	</td><td>		是否启用fetchKey上传模式。 0为关闭，1为启用。具体见callbackFetchKey详解。</td></tr>
        <tr><td>persistentOps </td><td> 	</td><td>		资源上传成功后触发执行的预转持久化处理指令列表。 每个指令是一个API规格字符串，多个指令用;分隔。 请参阅persistenOps详解与示例。同时添加persistentPipeline字段，使用专用队列处理，请参阅persistentPipeline。</td></tr>
        <tr><td>persistentNotifyUrl </td><td> 	</td><td>		接收持久化处理结果通知的URL。 必须是公网上可以正常进行POST请求并能响应"HTTP/1.1 200 OK"的有效URL。 该URL获取的内容和持久化处理状态查询(prefop)的处理结果一致。 发送body格式是`Content-Type`为`application/json`的POST请求，需要按照读取流的形式读取请求的body才能获取。</td></tr>
        <tr><td>persistentPipeline </td><td> 	</td><td>		转码队列名。 资源上传成功后，触发转码时指定独立的队列进行转码。为空则表示使用公用队列，处理速度比较慢。建议使用专用队列。</td></tr>
        <tr><td>saveKey </td><td> 	</td><td>		自定义资源名。 支持魔法变量及自定义变量。这个字段仅当用户上传的时候没有主动指定key的时候起作用。</td></tr>
        <tr><td>fsizeMin </td><td> 	</td><td>		限定上传文件大小最小值，单位：字节（Byte）。设置为k，即k及k以上的文件可以上传。</td></tr>
        <tr><td>fsizeLimit </td><td> 	</td><td>		限定上传文件大小最大值，单位：字节（Byte）。 超过限制上传文件大小的最大值会被判为上传失败，返回413状态码。</td></tr>
        <tr><td>detectMime </td><td> 	</td><td>		开启MimeType侦测功能。 设为非0值，则忽略上传端传递的文件MimeType信息，使用七牛服务器侦测内容后的判断结果。 默认设为0值，如上传端指定了MimeType则直接使用该值，否则按如下顺序侦测MimeType值： 1. 检查文件扩展名； 2. 检查Key扩展名； 3. 侦测内容。 如不能侦测出正确的值，会默认使用 `application/octet-stream` 。</td></tr>
        <tr><td>mimeLimit </td><td> 	</td><td>		限定用户上传的文件类型。 指定本字段值，七牛服务器会侦测文件内容以判断MimeType，再用判断值跟指定值进行匹配，匹配成功则允许上传，匹配失败则返回403状态码。 示例： ● `image/*`表示只允许上传图片类型  ● `image/jpeg;image/png`表示只允许上传jpg和png类型的图片  ● `!application/json;text/plain`表示禁止上传json文本和纯文本。注意最前面的感叹号！</td></tr>
        <tr><td>deleteAfterDays </td><td> 	</td><td>		文件在多少天后被删除，七牛将文件上传时间与指定的`deleteAfterDays`天数相加，得到的时间入到后一天的午夜(CST,中国标准时间)，从而得到文件删除开始时间。例如文件在2015年1月1日上午10:00 CST上传，指定`deleteAfterDays`为3天，那么会在2015年1月5日00:00 CST之后当天内删除文件。</td></tr>
        </table>

        如现有如下上传策略：
        ```php
        $putPolicy = array(
            'scope' => QINIU_BUCKET.':'.$filename ,
        	'deadline' => time()+3600 ,
        	'returnBody' => '{
        		"name": $(fname),
                "file_url": $(x:file_url)
        	}'
    	);
    	```
    	该上传策略定义了：
    	    1.  指定了图片上传至`QINIU_BUCKET`存储空间中，同时，该token只允许文件名为 `$filename` 的文件上传。
    	    2.  token有效时间为1个小时。
    	    3.  设置返回信息，返回上传的文件的文件名，和自定义参数中的 `file_url`

    2.  将上传策略序列化成为JSON格式：
        用户可以使用各种语言的 JSON 库，也可以手工拼接字符串。序列化后，应得到如下形式的字符串（字符串值以外部分不含空格或换行）：
        ```php
        $putPolicy = json_encode($putPolicy);
        ```
        或
        ```php
        $putPolicy = '{
            "scope": "my-bucket:sunflower.jpg",
            "deadline":1451491200,
            "returnBody":
                "{
                \"name\":$(fname),
                \"size\":$(fsize),
                \"w\":$(imageInfo.width),
                \"h\":$(imageInfo.height),
                \"hash\":$(etag)
            }"
        }'
        ```

    3.  对 JSON 编码的上传策略进行URL安全的Base64编码，得到待签名字符串：
        官方给的demo代码为：
        ```php
        encodedPutPolicy = urlsafe_base64_encode(putPolicy)
        ```
        运行之后，发现 `urlsafe_base64_encode` 这个函数是自定义的，估计相当于将 `+,/`号转换为 `-,_`
        ```php
        function _urlsafe_base64_encode($str){
    		$find = array('+', '/');
    		$replace = array('-', '_');
    		return str_replace($find, $replace, base64_encode($str));
    	}
	    ```

    4.  使用SecretKey对上一步生成的待签名字符串计算HMAC-SHA1签名：
        官方给的demo代码为：
        ```php
        sign = hmac_sha1(encodedPutPolicy, QINIU_SECRET_KEY)
        ```
        然而， `hmac_sha1` 这个函数也不是php自带的。经过搜索发现，其实PHP5.1.2之后的版本内置了直接产生的函数，只是名字不一样罢了： [hash_hmac](http://php.net/manual/en/function.hash-hmac.php)，因此需要将这里修改为：
        ```php
        $sign = hash_hmac('sha1' ,$encodedPutPolicy, QINIU_SECRET_KEY, true);
        ```
        第一个参数为哈希算法名（支持`md5`，`sha256`，`haval160,4`等，具体可到[ hash_algos()](http://php.net/manual/zh/function.hash-algos.php)中查询）；第二个参数为需要进行哈希的信息；第三个参数为秘钥；第四个参数为输出格式（true为输出二进制，false为输出16进制）

    5.  对签名进行URL安全的Base64编码：
        官方代码：
        ```
        encodedSign = urlsafe_base64_encode(sign)
        ```
        这里的 `urlsafe_base64_encode` 依然为上述的自定义函数。

    6.  将AccessKey、encodedSign 和 encodedPutPolicy 用英文符号 `:` 连接起来：
        ```
        $uploadToken = QINIU_ACCESS_KEY . ':' . $encodedSign . ':' . $encodedPutPolicy
        ```

    7.  返回token至客户端
        ```
        echo json_encode(array('token' => $uploadToken, 'key' => $filename, 'fileurl' => QINIU_DOMAIN));
        ```
        这里返回了文件名和文件url，主要是因为，保证文件在七牛中的唯一性，和可以随时更改七牛的空间访问地址。

3.  上传图片
    使用js直接上传图片至七牛服务器他的过程为：

    向服务器请求 `uploadToken` =>

    获取 'uploadToken` 后上传图片 =>

    上传成功，显示图片。

    **这里没有使用 zepto jquery 这种库，所以浏览器的兼容性为兼容 FormData 的现代浏览器**

    使用 xhr 和 FormData 进行异步传输数据。
    ```javascript
    function ajax(options){
        options.start && options.start.call('start');
        //执行上传操作
        var xhr = new XMLHttpRequest();
        xhr.open("post", options.url, true);
        xhr.setRequestHeader("X-Requested-With", "XMLHttpRequest");
        xhr.onreadystatechange = function() {
            if (xhr.readyState == 4) {
                returnDate = JSON.parse(xhr.responseText);
                options.success && options.success.call('success', returnDate);
            };
        };
        //表单数据
        var fd = new FormData();
        for (k in options.data) {
            fd.append(k, options.data[k]);
        }
        fd.append('file', options.file);
        //执行发送
        result = xhr.send(fd);
    }
    ```

    新建一个表单 form
    ```html
    <form action="" method="post" name="upload_form" hidden>
        <input type="file" name="file">
        <input type="hidden" name="key" value="">
        <input type="hidden" name="x:file_url" value="">
        <input type="hidden" name="token" value="">
    </form>
    ```

    给上传文件的按钮绑定一个 change 时间：
    ```javascript
    file.addEventListener('change', function(e){
        var selected_file = e.target.files[0];
        // 先请求服务器获取token
        ajax({
            url: '/upload.php',
            data: {
                filename: selected_file.name
            },
            start: function(){
                console.log('start to get uploadToken');
            },
            success: function(data){
                // 给表单中的参数赋值
                form.key.value = data.key;
                form.file_url.value = data.fileurl+data.key;
                form.token.value = data.token;

                // 执行上传图片操作
                uploadImage(selected_file)
            }
        })
    })
    ```

    开始上传文件至七牛
    ```javascript
    function uploadImage(file){
        ajax({
            // 如果
            url: 'http://upload.qiniu.com/',
            // url: 'https://up.qbox.me',
            data: {
                file: file,
                key: form.key.value,
                'x:file_url': form.file_url.value,
                token: form.token.value,
            },
            start: function(){
                console.log('start to upload Image to Qiniu');
            },
            success: function(data){
                // 给表单中的参数赋值
                console.log(data);
                image.src = data.file_url
            }
        })
    }
    ```

    最终实现：
    ![最终实现](http://ojrkbauy9.bkt.clouddn.com/973f9ed26b6ae914e6b7174c69a4bf6e)

    **[GitHub Demo](https://github.com/JZLeung/qiniu_upload_demo) 欢迎吐槽**
