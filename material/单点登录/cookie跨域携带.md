# @Configuration

```java
@Configuration
public class CORSConfiguration extends WebMvcConfigurerAdapter {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**").allowedOrigins("*")
                .allowedMethods("GET", "HEAD", "POST", "PUT", "DELETE", "OPTIONS")
                .allowCredentials(true).maxAge(3600);

    }
}
```



# index.html

```js
$(function(){
			$.ajax({
				//url: "http://sso.qf.com:9092/user/checkIsLogin",
				url:"checkIsLogin",
				type: 'get',
                //允许携带证书
                xhrFields: {
                    withCredentials: true
                },
                //允许跨域
                crossDomain: true,
				success: function(d){
					console.log(d);
					if(d.result==true){
						$("#loginBox").html("欢迎你，"+d.data.username+"!<a href=\"#\" target=\"_top\">注销</a>")
					}else{
						$("#loginBox").html("<a href=\"http://sso.qf.com:9092/user/login\" target=\"_top\" class=\"h\">亲，请登录</a>\n" +
								"\t\t<a href=\"#\" target=\"_top\">免费注册</a>");
					}
				}
			});
		})
```

