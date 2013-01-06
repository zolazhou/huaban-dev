---
title: 关于PHP的会话冲突
kind: article
author: Zola Zhou
created_at: 2011-03-19
published: true
image: 2011/03/expired-cookie.png
disqus_id: 'http://www.zolazhou.com/2011/03/something-about-php-session-security/'
---

发现PHP的一个会话(Session)安全性问题，这已经是去年的事了，当时就想写下来的，拖到现在是因为“我很忙”。
好吧，事实是我从小害怕写文字。最近有个朋友遇到同样的问题，跟他解释发生这种问题的原因时，
我有些说不太清楚，因为忘记了一些细节。所以，我决定今后有问题或想法还是得写下来。

本身接触PHP不久，不知道自己发现的这个问题是不是在圈子里已经是家喻户晓的事情，如果是，我承认我那朋友太凹凸了。

2010年6月份（用PHP构建的新版又拍网上线后不久），有一些用户反映自己登录到别人帐号里或是自己帐号里多了别人的照片。
显然，是会话被劫持了。但这应该不是一种主动的、恶意的劫持，不然不会有人告诉你，也不会只是传几张自己的照片在别人的相册里。
那么会不会是因为会话ID的生成方式太简单，出现重复的ID而导致会话冲突了呢？当时尝试了很多办法查找原因，
具体给忘了，你知道这是大半年前的事情了，我的记性也不好。 记得最后在跟踪了几个出现问题的用户日志后，
发现他们的会话ID都为`deleted`。

这个会话ID是通过cookie发送过来的，不知道你对这样的cookie是不是很熟悉。一开始以为是那些乱七八糟的浏览器外壳搞的鬼，
后来对自己的HTTP响应进行跟踪后，发现这个cookie尽然是我们自己设置的，在退出登录的HTTP请求响应里可以发现这样的头:

<pre><code class="language-c">
Set-Cookie: PHPSESSID=deleted; expires=Fri, 19-Mar-2010 07:20:51 GMT; path=/
</code></pre>

这个cookie的值为`deleted`，而过期时间是去年的这个时候再早1秒钟。这样的cookie仍然被浏览器发送到服务器上，
一个可能是用户的系统时间不正确，并在这个过期时间之前，这是很有可能的事情。因此有着同样问题的用户就一起共用了ID为
`deleted`的会话。当用户退出登录，我们执行了类似下面的这段代码

<pre><code class="language-php">
session_destroy(); // 消灭掉所有Session里的变量

$params = session_get_cookie_params();
// 再干掉保存session id的cookie
setcookie(session_name(), '', 1,
          $params['path'], $params['domain'],
          $params['secure'], $params['httponly']);
</code></pre>

给`setcookie`方法传递空字符串时，PHP会将这个cookie值替换为`deleted`，并且忽略指定的过期时间参数而直接输出为一年加一秒钟前。
在PHP的源代码里可以找这个逻辑的实现：

<pre><code class="language-c">
PHPAPI int php_setcookie(char *name, int name_len, char *value, int value_len, time_t expires, char *path, int path_len, char *domain, int domain_len, int secure, int url_encode, int httponly TSRMLS_DC)
    ...
    if (value && value_len == 0) {
        /* 
         * MSIE doesn't delete a cookie when you set it to a null value
         * so in order to force cookies to be deleted, even on MSIE, we
         * pick an expiry date 1 year and 1 second in the past
         */
        time_t t = time(NULL) - 31536001;
        dt = php_format_date("D, d-M-Y H:i:s T", sizeof("D, d-M-Y H:i:s T")-1, t, 0 TSRMLS_CC);
        snprintf(cookie, len + 100, "Set-Cookie: %s=deleted; expires=%s", name, dt);
        efree(dt);
    } else {
        snprintf(cookie, len + 100, "Set-Cookie: %s=%s", name, value ? encoded_value : "");
        if (expires > 0) {
            strlcat(cookie, "; expires=", len + 100);
            dt = php_format_date("D, d-M-Y H:i:s T", sizeof("D, d-M-Y H:i:s T")-1, expires, 0 TSRMLS_CC);
            strlcat(cookie, dt, len + 100);
            efree(dt);
        }
    }
</code></pre>

发生这种问题的用户一定是清晰的退出登录过，而之后又回到网站就开始使用了`deleted`为ID的会话。
我们在调用`session_start`方法开启会话时，PHP检查cookie里的会话ID，如果存在那么就用那个ID去会话存储里（本地文件系统、数据库、memcached不管是什么）
作为key获取或创建一个会话，PHP不会检查这个ID值是什么样子的。

问题已经清楚了，那么我们要怎么避免这种问题的发生呢？

* 首先使用安全的ID生成算法，在开启会话时，进行会话ID值的有效性判断。如果用户不需要登录你的网站就能作一些会话相关操作时，
这个方法可以结合下面的方法使用。

* 在会话中保存用户浏览器(User-Agent)信息，在开启会话时进行一致性检查。不建议使用IP地址的一致性检查，
这个做法会给正常使用的用户带来很大程度的影响，很多用户的IP会在会话期间发生变化。

* 在用户登录时(清晰的、或自动登录)，调用`session_regenerate_id`更改会话ID，这是最为有效的方法。可以防止恶意的会话劫持攻击。

* 最后，使用下面的方式退出登录:

<pre><code class="language-php">
session_destroy(); // 消灭掉所有Session里的变量

$params = session_get_cookie_params();
// 再干掉保存session id的cookie
setcookie(session_name(), session_id(), 1,
          $params['path'], $params['domain'],
          $params['secure'], $params['httponly']);

// all is well
</code></pre>

这里我们用当前会话ID替代空字符串作为cookie值，这样一来，就算这个cookie再被浏览器发送回来，该用户顶多再次退出前的会话，
而不会和别的用户冲突。而当他再登录时，我们又会使用`session_regenerate_id`为他生成新的会话ID。
过期时间我们指定为1(Thu, 01-Jan-1970 00:00:01 GMT)，而不是0，因为指定为0的话代表cookie的有效期会会话（关闭浏览器前），
在HTTP Set-Cookie头里不会有expires部分。

题外话
---------

如果你使用[memcached][]作为会话存储实现，那么你要注意这个PHP扩展里存在一个BUG对我们讨论的问题的发生起到一些催化作用。
这个BUG会导致会话过期时间无效，也就是说除非memcached内存使用量达到限额，长期未访问的会话对象才有可能因为被删除而失效。
会话的长期有效会增加有问题的用户的碰撞可能性。

问题出在这个扩展在获取ini配置的代码:

<pre><code class="language-c">
sess_lifetime = zend_ini_long(ZEND_STRL("session.gc_maxlifetime"), 0);

if (sess_lifetime > 0) {
    expiration = time(NULL) + sess_lifetime;
} else {
    expiration = 0;
}
</code></pre>

`zend_ini_long`方法用于读取PHP的配置项，第一个参数为配置项名称，第二个参数为配置项名称字符串长度，必须包括字符串终止符号'\0'，
第三个参数是默认值。而ZEND_STRL这个宏定义返回的字符串长度并不包含终止符号，所以这个调用实际上每次都只会返回0。
而对于memcached来说过期时间为0代表不会过期。我们只要用ZEND_STRS宏替换ZEND_STRL就可以了。

另外，当时我还发现[memcached][]这个扩展的一个内存泄漏问题，也与会话有关。当我们使用的memcached服务器越多，泄漏越明显。
因为作者没有释放`memcached_server_st`结构:

<pre><code class="language-c">
PS_OPEN_FUNC(memcached)
{
    memcached_st *memc_sess = PS_GET_MOD_DATA();
    memcached_server_st *servers;
    memcached_return status;

    servers = memcached_servers_parse((char *)save_path);
    if (servers) {
        memc_sess = memcached_create(NULL);
        if (memc_sess) {
            status = memcached_server_push(memc_sess, servers);
            if (status == MEMCACHED_SUCCESS) {
                PS_SET_MOD_DATA(memc_sess);
                // Zola: 这里必须释放内存
                memcached_server_list_free(servers);
                // end
                return SUCCESS;
            }    
        } else {
            php_error_docref(NULL TSRMLS_CC, E_WARNING, "could not allocate libmemcached structure");
        }    
        // Zola: 这里必须释放内存
        memcached_server_list_free(servers);
        // end
    } else {
        php_error_docref(NULL TSRMLS_CC, E_WARNING, "failed to parse session.save_path");
    }    

    PS_SET_MOD_DATA(NULL);
    return FAILURE;
}
</code></pre>

写这篇文章的时候去看了一下[memcached][]的最新代码，这两个BUG已经在最新版本(2.0.0b1)里被修复。如果你还是使用老的版本，
那么升级吧，或自己调整一下代码。


总结一下
------------

做了这么多年的Web开发，一直没有对session、cookie、过期之类的问题做过深入的了解，有些东西认为很基础，我们可能会轻易忽略掉。
希望这篇文章能对新的PHP开发者有所帮助。


[memcached]: http://pecl.php.net/package/memcached
