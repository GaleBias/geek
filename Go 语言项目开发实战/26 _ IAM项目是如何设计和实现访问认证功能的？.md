<audio title="26 _ IAM项目是如何设计和实现访问认证功能的？" src="https://static001.geekbang.org/resource/audio/09/f9/095f81d524a2ed3541177750768146f9.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>上一讲，我们学习了应用认证常用的四种方式：Basic、Digest、OAuth、Bearer。这一讲，我们再来看下IAM项目是如何设计和实现认证功能的。</p><p>IAM项目用到了Basic认证和Bearer认证。其中，Basic认证用在前端登陆的场景，Bearer认证用在调用后端API服务的场景下。</p><p>接下来，我们先来看下IAM项目认证功能的整体设计思路。</p><h2>如何设计IAM项目的认证功能？</h2><p>在认证功能开发之前，我们要根据需求，认真考虑下如何设计认证功能，并在设计阶段通过技术评审。那么我们先来看下，如何设计IAM项目的认证功能。</p><p>首先，我们要<strong>梳理清楚认证功能的使用场景和需求</strong>。</p><ul>
<li>IAM项目的iam-apiserver服务，提供了IAM系统的管理流功能接口，它的客户端可以是前端（这里也叫控制台），也可以是App端。</li>
<li>为了方便用户在Linux系统下调用，IAM项目还提供了iamctl命令行工具。</li>
<li>为了支持在第三方代码中调用iam-apiserver提供的API接口，还支持了API调用。</li>
<li>为了提高用户在代码中调用API接口的效率，IAM项目提供了Go SDK。</li>
</ul><p>可以看到，iam-apiserver有很多客户端，每种客户端适用的认证方式是有区别的。</p><!-- [[[read_end]]] --><p>控制台、App端需要登录系统，所以需要使用<code>用户名：密码</code>这种认证方式，也即Basic认证。iamctl、API调用、Go SDK因为可以不用登录系统，所以可以采用更安全的认证方式：Bearer认证。同时，Basic认证作为iam-apiserver已经集成的认证方式，仍然可以供iamctl、API调用、Go SDK使用。</p><p>这里有个地方需要注意：如果iam-apiserver采用Bearer Token的认证方式，目前最受欢迎的Token格式是JWT Token。而JWT Token需要密钥（后面统一用secretKey来指代），因此需要在iam-apiserver服务中为每个用户维护一个密钥，这样会增加开发和维护成本。</p><p>业界有一个更好的实现方式：将iam-apiserver提供的API接口注册到API网关中，通过API网关中的Token认证功能，来实现对iam-apiserver API接口的认证。有很多API网关可供选择，例如腾讯云API网关、Tyk、Kong等。</p><p>这里需要你注意：通过iam-apiserver创建的密钥对是提供给iam-authz-server使用的。</p><p>另外，我们还需要调用iam-authz-server提供的RESTful API接口：<code>/v1/authz</code>，来进行资源授权。API调用比较适合采用的认证方式是Bearer认证。</p><p>当然，<code>/v1/authz</code>也可以直接注册到API网关中。在实际的Go项目开发中，也是我推荐的一种方式。但在这里，为了展示实现Bearer认证的过程，iam-authz-server自己实现了Bearer认证。讲到iam-authz-server Bearer认证实现的时候，我会详细介绍这一点。</p><p>Basic认证需要用户名和密码，Bearer认证则需要密钥，所以iam-apiserver需要将用户名/密码、密钥等信息保存在后端的MySQL中，持久存储起来。</p><p>在进行认证的时候，需要获取密码或密钥进行反加密，这就需要查询密码或密钥。查询密码或密钥有两种方式。一种是在请求到达时查询数据库。因为数据库的查询操作延时高，会导致API接口延时较高，所以不太适合用在数据流组件中。另外一种是将密码或密钥缓存在内存中，这样请求到来时，就可以直接从内存中查询，从而提升查询速度，提高接口性能。</p><p>但是，将密码或密钥缓存在内存中时，就要考虑内存和数据库的数据一致性，这会增加代码实现的复杂度。因为管控流组件对性能延时要求不那么敏感，而数据流组件则一定要实现非常高的接口性能，所以iam-apiserver在请求到来时查询数据库，而iam-authz-server则将密钥信息缓存在内存中。</p><p>那在这里，可以总结出一张IAM项目的认证设计图：</p><p><img src="https://static001.geekbang.org/resource/image/7e/b6/7eed8e2364d358a8483c671d972fd2b6.jpg?wh=2248x1094" alt=""></p><p>另外，为了将控制流和数据流区分开来，密钥的CURD操作也放在了iam-apiserver中，但是iam-authz-server需要用到这些密钥信息。为了解决这个问题，目前的做法是：</p><ul>
<li>iam-authz-server通过gRPC API请求iam-apiserver，获取所有的密钥信息；</li>
<li>当iam-apiserver有密钥更新时，会Pub一条消息到Redis Channel中。因为iam-authz-server订阅了同一个Redis Channel，iam-authz-searver监听到channel有新消息时，会获取、解析消息，并更新它缓存的密钥信息。这样，我们就能确保iam-authz-server内存中缓存的密钥和iam-apiserver中的密钥保持一致。</li>
</ul><p>学到这里，你可能会问：将所有密钥都缓存在iam-authz-server中，那岂不是要占用很大的内存？别担心，这个问题我也想过，并且替你计算好了：8G的内存大概能保存约8千万个密钥信息，完全够用。后期不够用的话，可以加大内存。</p><p>不过这里还是有个小缺陷：如果Redis down掉，或者出现网络抖动，可能会造成iam-apiserver中和iam-authz-server内存中保存的密钥数据不一致，但这不妨碍我们学习认证功能的设计和实现。至于如何保证缓存系统的数据一致性，我会在新一期的特别放送里专门介绍下。</p><p>最后注意一点：Basic 认证请求和 Bearer 认证请求都可能被截获并重放。所以，为了确保Basic认证和Bearer认证的安全性，<strong>和服务端通信时都需要配合使用HTTPS协议</strong>。</p><h2>IAM项目是如何实现Basic认证的？</h2><p>我们已经知道，IAM项目中主要用了Basic 和 Bearer 这两种认证方式。我们要支持Basic认证和Bearer认证，并根据需要选择不同的认证方式，这很容易让我们想到使用设计模式中的策略模式来实现。所以，在IAM项目中，我将每一种认证方式都视作一个策略，通过选择不同的策略，来使用不同的认证方法。</p><p>IAM项目实现了如下策略：</p><ul>
<li><a href="https://github.com/marmotedu/iam/blob/v1.0.0/internal/pkg/middleware/auth/auto.go">auto策略</a>：该策略会根据HTTP头<code>Authorization: Basic XX.YY.ZZ</code>和<code>Authorization: Bearer XX.YY.ZZ</code>自动选择使用Basic认证还是Bearer认证。</li>
<li><a href="https://github.com/marmotedu/iam/blob/v1.0.0/internal/pkg/middleware/auth/basic.go">basic策略</a>：该策略实现了Basic认证。</li>
<li><a href="https://github.com/marmotedu/iam/blob/v1.0.0/internal/pkg/middleware/auth/jwt.go">jwt策略</a>：该策略实现了Bearer认证，JWT是Bearer认证的具体实现。</li>
<li><a href="https://github.com/marmotedu/iam/blob/v1.0.0/internal/pkg/middleware/auth/cache.go">cache策略</a>：该策略其实是一个Bearer认证的实现，Token采用了JWT格式，因为Token中的密钥ID是从内存中获取的，所以叫Cache认证。这一点后面会详细介绍。</li>
</ul><p>iam-apiserver通过创建需要的认证策略，并加载到需要认证的API路由上，来实现API认证。具体代码如下：</p><pre><code>jwtStrategy, _ := newJWTAuth().(auth.JWTStrategy)
g.POST(&quot;/login&quot;, jwtStrategy.LoginHandler)
g.POST(&quot;/logout&quot;, jwtStrategy.LogoutHandler)
// Refresh time can be longer than token timeout
g.POST(&quot;/refresh&quot;, jwtStrategy.RefreshHandler)
</code></pre><p>上述代码中，我们通过<a href="https://github.com/marmotedu/iam/blob/75b978b722f0af3d6aefece3f9668269be3f5b2e/internal/apiserver/auth.go#L59">newJWTAuth</a>函数创建了<code>auth.JWTStrategy</code>类型的变量，该变量包含了一些认证相关函数。</p><ul>
<li>LoginHandler：实现了Basic认证，完成登陆认证。</li>
<li>RefreshHandler：重新刷新Token的过期时间。</li>
<li>LogoutHandler：用户注销时调用。登陆成功后，如果在Cookie中设置了认证相关的信息，执行LogoutHandler则会清空这些信息。</li>
</ul><p>下面，我来分别介绍下LoginHandler、RefreshHandler和LogoutHandler。</p><ol>
<li>LoginHandler</li>
</ol><p>这里，我们来看下LoginHandler Gin中间件，该函数定义位于<code>github.com/appleboy/gin-jwt</code>包的<a href="https://github.com/appleboy/gin-jwt/blob/v2.6.4/auth_jwt.go#L431">auth_jwt.go</a>文件中。</p><pre><code>func (mw *GinJWTMiddleware) LoginHandler(c *gin.Context) {
	if mw.Authenticator == nil {
		mw.unauthorized(c, http.StatusInternalServerError, mw.HTTPStatusMessageFunc(ErrMissingAuthenticatorFunc, c))
		return
	}

	data, err := mw.Authenticator(c)

	if err != nil {
		mw.unauthorized(c, http.StatusUnauthorized, mw.HTTPStatusMessageFunc(err, c))
		return
	}

	// Create the token
	token := jwt.New(jwt.GetSigningMethod(mw.SigningAlgorithm))
	claims := token.Claims.(jwt.MapClaims)

	if mw.PayloadFunc != nil {
		for key, value := range mw.PayloadFunc(data) {
			claims[key] = value
		}
	}

	expire := mw.TimeFunc().Add(mw.Timeout)
	claims[&quot;exp&quot;] = expire.Unix()
	claims[&quot;orig_iat&quot;] = mw.TimeFunc().Unix()
	tokenString, err := mw.signedString(token)

	if err != nil {
		mw.unauthorized(c, http.StatusUnauthorized, mw.HTTPStatusMessageFunc(ErrFailedTokenCreation, c))
		return
	}

	// set cookie
	if mw.SendCookie {
		expireCookie := mw.TimeFunc().Add(mw.CookieMaxAge)
		maxage := int(expireCookie.Unix() - mw.TimeFunc().Unix())

		if mw.CookieSameSite != 0 {
			c.SetSameSite(mw.CookieSameSite)
		}

		c.SetCookie(
			mw.CookieName,
			tokenString,
			maxage,
			&quot;/&quot;,
			mw.CookieDomain,
			mw.SecureCookie,
			mw.CookieHTTPOnly,
		)
	}

	mw.LoginResponse(c, http.StatusOK, tokenString, expire)
}
</code></pre><p>从LoginHandler函数的代码实现中，我们可以知道，LoginHandler函数会执行<code>Authenticator</code>函数，来完成Basic认证。如果认证通过，则会签发JWT Token，并执行 <code>PayloadFunc</code>函数设置Token Payload。如果我们设置了 <code>SendCookie=true</code> ，还会在Cookie中添加认证相关的信息，例如 Token、Token的生命周期等，最后执行 <code>LoginResponse</code> 方法返回Token和Token的过期时间。</p><p><code>Authenticator</code>、<code>PayloadFunc</code>、<code>LoginResponse</code>这三个函数，是我们在创建JWT认证策略时指定的。下面我来分别介绍下。</p><p>先来看下<a href="https://github.com/marmotedu/iam/blob/v1.0.0/internal/apiserver/auth.go#L97">Authenticator</a>函数。Authenticator函数从HTTP Authorization Header中获取用户名和密码，并校验密码是否合法。</p><pre><code>func authenticator() func(c *gin.Context) (interface{}, error) {
	return func(c *gin.Context) (interface{}, error) {
		var login loginInfo
		var err error

		// support header and body both
		if c.Request.Header.Get(&quot;Authorization&quot;) != &quot;&quot; {
			login, err = parseWithHeader(c)
		} else {
			login, err = parseWithBody(c)
		}
		if err != nil {
			return &quot;&quot;, jwt.ErrFailedAuthentication
		}

		// Get the user information by the login username.
		user, err := store.Client().Users().Get(c, login.Username, metav1.GetOptions{})
		if err != nil {
			log.Errorf(&quot;get user information failed: %s&quot;, err.Error())

			return &quot;&quot;, jwt.ErrFailedAuthentication
		}

		// Compare the login password with the user password.
		if err := user.Compare(login.Password); err != nil {
			return &quot;&quot;, jwt.ErrFailedAuthentication
		}

		return user, nil
	}
}
</code></pre><p><code>Authenticator</code>函数需要获取用户名和密码。它首先会判断是否有<code>Authorization</code>请求头，如果有，则调用<code>parseWithHeader</code>函数获取用户名和密码，否则调用<code>parseWithBody</code>从Body中获取用户名和密码。如果都获取失败，则返回认证失败错误。</p><p>所以，IAM项目的Basic支持以下两种请求方式：</p><pre><code>$ curl -XPOST -H&quot;Authorization: Basic YWRtaW46QWRtaW5AMjAyMQ==&quot; http://127.0.0.1:8080/login # 用户名:密码通过base64加码后，通过HTTP Authorization Header进行传递，因为密码非明文，建议使用这种方式。
$ curl -s -XPOST -H'Content-Type: application/json' -d'{&quot;username&quot;:&quot;admin&quot;,&quot;password&quot;:&quot;Admin@2021&quot;}' http://127.0.0.1:8080/login # 用户名和密码在HTTP Body中传递，因为密码是明文，所以这里不建议实际开发中，使用这种方式。
</code></pre><p>这里，我们来看下 <code>parseWithHeader</code> 是如何获取用户名和密码的。假设我们的请求为：</p><pre><code>$ curl -XPOST -H&quot;Authorization: Basic YWRtaW46QWRtaW5AMjAyMQ==&quot; http://127.0.0.1:8080/login
</code></pre><p>其中，<code>YWRtaW46QWRtaW5AMjAyMQ==</code>值由以下命令生成：</p><pre><code>$ echo -n 'admin:Admin@2021'|base64
YWRtaW46QWRtaW5AMjAyMQ==
</code></pre><p><code>parseWithHeader</code>实际上执行的是上述命令的逆向步骤：</p><ol>
<li>获取<code>Authorization</code>头的值，并调用strings.SplitN函数，获取一个切片变量auth，其值为 <code>["Basic","YWRtaW46QWRtaW5AMjAyMQ=="]</code> 。</li>
<li>将<code>YWRtaW46QWRtaW5AMjAyMQ==</code>进行base64解码，得到<code>admin:Admin@2021</code>。</li>
<li>调用<code>strings.SplitN</code>函数获取 <code>admin:Admin@2021</code> ，得到用户名为<code>admin</code>，密码为<code>Admin@2021</code>。</li>
</ol><p><code>parseWithBody</code>则是调用了Gin的<code>ShouldBindJSON</code>函数，来从Body中解析出用户名和密码。</p><p>获取到用户名和密码之后，程序会从数据库中查询出该用户对应的加密后的密码，这里我们假设是<code>xxxx</code>。最后<code>authenticator</code>函数调用<code>user.Compare</code>来判断 <code>xxxx</code> 是否和通过<code>user.Compare</code>加密后的字符串相匹配，如果匹配则认证成功，否则返回认证失败。</p><p>再来看下<code>PayloadFunc</code>函数：</p><pre><code>func payloadFunc() func(data interface{}) jwt.MapClaims {
    return func(data interface{}) jwt.MapClaims {
        claims := jwt.MapClaims{
            &quot;iss&quot;: APIServerIssuer,
            &quot;aud&quot;: APIServerAudience,
        }
        if u, ok := data.(*v1.User); ok {
            claims[jwt.IdentityKey] = u.Name
            claims[&quot;sub&quot;] = u.Name
        }

        return claims
    }
}
</code></pre><p>PayloadFunc函数会设置JWT Token中Payload部分的 iss、aud、sub、identity字段，供后面使用。</p><p>再来看下我们刚才说的第三个函数，LoginResponse函数：</p><pre><code>func loginResponse() func(c *gin.Context, code int, token string, expire time.Time) {
    return func(c *gin.Context, code int, token string, expire time.Time) {
        c.JSON(http.StatusOK, gin.H{
            &quot;token&quot;:  token,
            &quot;expire&quot;: expire.Format(time.RFC3339),
        })
    }
}
</code></pre><p>该函数用来在Basic认证成功之后，返回Token和Token的过期时间给调用者：</p><pre><code>$ curl -XPOST -H&quot;Authorization: Basic YWRtaW46QWRtaW5AMjAyMQ==&quot; http://127.0.0.1:8080/login
{&quot;expire&quot;:&quot;2021-09-29T01:38:49+08:00&quot;,&quot;token&quot;:&quot;XX.YY.ZZ&quot;}
</code></pre><p>登陆成功后，iam-apiserver会返回Token和Token的过期时间，前端可以将这些信息缓存在Cookie中或LocalStorage中，之后的请求都可以使用Token来进行认证。使用Token进行认证，不仅能够提高认证的安全性，还能够避免查询数据库，从而提高认证效率。</p><ol start="2">
<li>RefreshHandler</li>
</ol><p><code>RefreshHandler</code>函数会先执行Bearer认证，如果认证通过，则会重新签发Token。</p><ol start="3">
<li>LogoutHandler</li>
</ol><p>最后，来看下<code>LogoutHandler</code>函数：</p><pre><code>func (mw *GinJWTMiddleware) LogoutHandler(c *gin.Context) {
    // delete auth cookie
    if mw.SendCookie {
        if mw.CookieSameSite != 0 {
            c.SetSameSite(mw.CookieSameSite)
        }

        c.SetCookie(
            mw.CookieName,
            &quot;&quot;,
            -1,
            &quot;/&quot;,
            mw.CookieDomain,
            mw.SecureCookie,
            mw.CookieHTTPOnly,
        )
    }

    mw.LogoutResponse(c, http.StatusOK)
}
</code></pre><p>可以看到，LogoutHandler其实是用来清空Cookie中Bearer认证相关信息的。</p><p>最后，我们来做个总结：Basic认证通过用户名和密码来进行认证，通常用在登陆接口/login中。用户登陆成功后，会返回JWT Token，前端会保存该JWT Token在浏览器的Cookie或LocalStorage中，供后续请求使用。</p><p>后续请求时，均会携带该Token，以完成Bearer认证。另外，有了登陆接口，一般还会配套/logout接口和/refresh接口，分别用来进行注销和刷新Token。</p><p>这里你可能会问，为什么要刷新Token？因为通过登陆接口签发的Token有过期时间，有了刷新接口，前端就可以根据需要，自行刷新Token的过期时间。过期时间可以通过iam-apiserver配置文件的<a href="https://github.com/marmotedu/iam/blob/master/configs/iam-apiserver.yaml#L66">jwt.timeout</a>配置项来指定。登陆后签发Token时，使用的密钥（secretKey）由<a href="https://github.com/marmotedu/iam/blob/master/configs/iam-apiserver.yaml#L65">jwt.key</a>配置项来指定。</p><h2>IAM项目是如何实现Bearer认证的？</h2><p>上面我们介绍了Basic认证。这里，我再来介绍下IAM项目中Bearer认证的实现方式。</p><p>IAM项目中有两个地方实现了Bearer认证，分别是 iam-apiserver 和 iam-authz-server。下面我来分别介绍下它们是如何实现Bearer认证的。</p><h3>iam-authz-server Bearer认证实现</h3><p>先来看下iam-authz-server是如何实现Bearer认证的。</p><p>iam-authz-server通过在 <code>/v1</code> 路由分组中加载cache认证中间件来使用cache认证策略：</p><pre><code>auth := newCacheAuth()
apiv1 := g.Group(&quot;/v1&quot;, auth.AuthFunc())
</code></pre><p>来看下<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/authzserver/jwt.go#L15">newCacheAuth</a>函数：</p><pre><code>func newCacheAuth() middleware.AuthStrategy {
    return auth.NewCacheStrategy(getSecretFunc())
}

func getSecretFunc() func(string) (auth.Secret, error) {
    return func(kid string) (auth.Secret, error) {
        cli, err := store.GetStoreInsOr(nil)
        if err != nil {
            return auth.Secret{}, errors.Wrap(err, &quot;get store instance failed&quot;)
        }

        secret, err := cli.GetSecret(kid)
        if err != nil {
            return auth.Secret{}, err
        }

        return auth.Secret{
            Username: secret.Username,
            ID:       secret.SecretId,
            Key:      secret.SecretKey,
            Expires:  secret.Expires,
        }, nil
    }
}
</code></pre><p>newCacheAuth函数调用<code>auth.NewCacheStrategy</code>创建了一个cache认证策略，创建时传入了<code>getSecretFunc</code>函数，该函数会返回密钥的信息。密钥信息包含了以下字段：</p><pre><code>type Secret struct {
    Username string
    ID       string
    Key      string
    Expires  int64
}
</code></pre><p>再来看下cache认证策略实现的<a href="https://github.com/marmotedu/iam/blob/master/internal/pkg/middleware/auth/cache.go#L48">AuthFunc</a>方法：</p><pre><code>func (cache CacheStrategy) AuthFunc() gin.HandlerFunc {
	return func(c *gin.Context) {
		header := c.Request.Header.Get(&quot;Authorization&quot;)
		if len(header) == 0 {
			core.WriteResponse(c, errors.WithCode(code.ErrMissingHeader, &quot;Authorization header cannot be empty.&quot;), nil)
			c.Abort()

			return
		}

		var rawJWT string
		// Parse the header to get the token part.
		fmt.Sscanf(header, &quot;Bearer %s&quot;, &amp;rawJWT)

		// Use own validation logic, see below
		var secret Secret

		claims := &amp;jwt.MapClaims{}
		// Verify the token
		parsedT, err := jwt.ParseWithClaims(rawJWT, claims, func(token *jwt.Token) (interface{}, error) {
			// Validate the alg is HMAC signature
			if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf(&quot;unexpected signing method: %v&quot;, token.Header[&quot;alg&quot;])
			}

			kid, ok := token.Header[&quot;kid&quot;].(string)
			if !ok {
				return nil, ErrMissingKID
			}

			var err error
			secret, err = cache.get(kid)
			if err != nil {
				return nil, ErrMissingSecret
			}

			return []byte(secret.Key), nil
		}, jwt.WithAudience(AuthzAudience))
		if err != nil || !parsedT.Valid {
			core.WriteResponse(c, errors.WithCode(code.ErrSignatureInvalid, err.Error()), nil)
			c.Abort()

			return
		}

		if KeyExpired(secret.Expires) {
			tm := time.Unix(secret.Expires, 0).Format(&quot;2006-01-02 15:04:05&quot;)
			core.WriteResponse(c, errors.WithCode(code.ErrExpired, &quot;expired at: %s&quot;, tm), nil)
			c.Abort()

			return
		}

		c.Set(CtxUsername, secret.Username)
		c.Next()
	}
}

// KeyExpired checks if a key has expired, if the value of user.SessionState.Expires is 0, it will be ignored.
func KeyExpired(expires int64) bool {
	if expires &gt;= 1 {
		return time.Now().After(time.Unix(expires, 0))
	}

	return false
}
</code></pre><p>AuthFunc函数依次执行了以下四大步来完成JWT认证，每一步中又有一些小步骤，下面我们来一起看看。</p><p>第一步，从Authorization: Bearer XX.YY.ZZ请求头中获取XX.YY.ZZ，XX.YY.ZZ即为JWT Token。</p><p>第二步，调用github.com/dgrijalva/jwt-go包提供的ParseWithClaims函数，该函数会依次执行下面四步操作。</p><p>调用ParseUnverified函数，依次执行以下操作：</p><p>从Token中获取第一段XX，base64解码后得到JWT Token的Header{“alg”:“HS256”,“kid”:“a45yPqUnQ8gljH43jAGQdRo0bXzNLjlU0hxa”,“typ”:“JWT”}。</p><p>从Token中获取第二段YY，base64解码后得到JWT Token的Payload{“aud”:“iam.authz.marmotedu.com”,“exp”:1625104314,“iat”:1625097114,“iss”:“iamctl”,“nbf”:1625097114}。</p><p>根据Token Header中的alg字段，获取Token加密函数。</p><p>最终ParseUnverified函数会返回Token类型的变量，Token类型包含 Method、Header、Claims、Valid这些重要字段，这些字段会用于后续的认证步骤中。</p><p>调用传入的keyFunc获取密钥，这里来看下keyFunc的实现：</p><pre><code>func(token *jwt.Token) (interface{}, error) {
	// Validate the alg is HMAC signature
	if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
		return nil, fmt.Errorf(&quot;unexpected signing method: %v&quot;, token.Header[&quot;alg&quot;])
	}

	kid, ok := token.Header[&quot;kid&quot;].(string)
	if !ok {
		return nil, ErrMissingKID
	}

	var err error
	secret, err = cache.get(kid)
	if err != nil {
		return nil, ErrMissingSecret
	}

	return []byte(secret.Key), nil
}
</code></pre><p>可以看到，keyFunc接受 <code>*Token</code> 类型的变量，并获取Token Header中的kid，kid即为密钥ID：secretID。接着，调用cache.get(kid)获取密钥secretKey。cache.get函数即为getSecretFunc，getSecretFunc函数会根据kid，从内存中查找密钥信息，密钥信息中包含了secretKey。</p><ol start="3">
<li>从Token中获取Signature签名字符串ZZ，也即Token的第三段。</li>
<li>获取到secretKey之后，token.Method.Verify验证Signature签名字符串ZZ，也即Token的第三段是否合法。token.Method.Verify实际上是使用了相同的加密算法和相同的secretKey加密XX.YY字符串。假设加密之后的字符串为WW，接下来会用WW和ZZ base64解码后的字符串进行比较，如果相等则认证通过，如果不相等则认证失败。</li>
</ol><p><strong>第三步，</strong>调用KeyExpired，验证secret是否过期。secret信息中包含过期时间，你只需要拿该过期时间和当前时间对比就行。</p><p><strong>第四步，</strong>设置HTTP Header<code>username: colin</code>。</p><p>到这里，iam-authz-server的Bearer认证分析就完成了。</p><p>我们来做个总结：iam-authz-server通过加载Gin中间件的方式，在请求<code>/v1/authz</code>接口时进行访问认证。因为Bearer认证具有过期时间，而且可以在认证字符串中携带更多有用信息，还具有不可逆加密等优点，所以<strong>/v1/authz采用了Bearer认证，Token格式采用了JWT格式</strong>，这也是业界在API认证中最受欢迎的认证方式。</p><p>Bearer认证需要secretID和secretKey，这些信息会通过gRPC接口调用，从iam-apisaerver中获取，并缓存在iam-authz-server的内存中供认证时查询使用。</p><p>当请求来临时，iam-authz-server Bearer认证中间件从JWT Token中解析出Header，并从Header的kid字段中获取到secretID，根据secretID查找到secretKey，最后使用secretKey加密JWT Token的Header和Payload，并与Signature部分进行对比。如果相等，则认证通过；如果不等，则认证失败。</p><h3>iam-apiserver Bearer认证实现</h3><p>再来看下 iam-apiserver的Bearer认证。</p><p>iam-apiserver的Bearer认证通过以下代码（位于<a href="https://github.com/marmotedu/iam/blob/v1.1.0/internal/apiserver/router.go#L65">router.go</a>文件中）指定使用了auto认证策略：</p><pre><code>v1.Use(auto.AuthFunc())
</code></pre><p>我们来看下<a href="https://github.com/marmotedu/iam/blob/v1.0.0/internal/pkg/middleware/auth/auto.go#L38">auto.AuthFunc()</a>的实现：</p><pre><code>func (a AutoStrategy) AuthFunc() gin.HandlerFunc {
	return func(c *gin.Context) {
		operator := middleware.AuthOperator{}
		authHeader := strings.SplitN(c.Request.Header.Get(&quot;Authorization&quot;), &quot; &quot;, 2)

		if len(authHeader) != authHeaderCount {
			core.WriteResponse(
				c,
				errors.WithCode(code.ErrInvalidAuthHeader, &quot;Authorization header format is wrong.&quot;),
				nil,
			)
			c.Abort()

			return
		}

		switch authHeader[0] {
		case &quot;Basic&quot;:
			operator.SetStrategy(a.basic)
		case &quot;Bearer&quot;:
			operator.SetStrategy(a.jwt)
			// a.JWT.MiddlewareFunc()(c)
		default:
			core.WriteResponse(c, errors.WithCode(code.ErrSignatureInvalid, &quot;unrecognized Authorization header.&quot;), nil)
			c.Abort()

			return
		}

		operator.AuthFunc()(c)

		c.Next()
	}
}
</code></pre><p>从上面代码中可以看到，AuthFunc函数会从Authorization Header中解析出认证方式是Basic还是Bearer。如果是Bearer，就会使用JWT认证策略；如果是Basic，就会使用Basic认证策略。</p><p>我们再来看下JWT认证策略的<a href="https://github.com/marmotedu/iam/blob/v1.0.0/internal/pkg/middleware/auth/jwt.go#L30">AuthFunc</a>函数实现：</p><pre><code>func (j JWTStrategy) AuthFunc() gin.HandlerFunc {
	return j.MiddlewareFunc()
}
</code></pre><p>我们跟随代码，可以定位到<code>MiddlewareFunc</code>函数最终调用了<code>github.com/appleboy/gin-jwt</code>包<code>GinJWTMiddleware</code>结构体的<a href="https://github.com/appleboy/gin-jwt/blob/v2.6.4/auth_jwt.go#L369">middlewareImpl</a>方法：</p><pre><code>func (mw *GinJWTMiddleware) middlewareImpl(c *gin.Context) {
	claims, err := mw.GetClaimsFromJWT(c)
	if err != nil {
		mw.unauthorized(c, http.StatusUnauthorized, mw.HTTPStatusMessageFunc(err, c))
		return
	}

	if claims[&quot;exp&quot;] == nil {
		mw.unauthorized(c, http.StatusBadRequest, mw.HTTPStatusMessageFunc(ErrMissingExpField, c))
		return
	}

	if _, ok := claims[&quot;exp&quot;].(float64); !ok {
		mw.unauthorized(c, http.StatusBadRequest, mw.HTTPStatusMessageFunc(ErrWrongFormatOfExp, c))
		return
	}

	if int64(claims[&quot;exp&quot;].(float64)) &lt; mw.TimeFunc().Unix() {
		mw.unauthorized(c, http.StatusUnauthorized, mw.HTTPStatusMessageFunc(ErrExpiredToken, c))
		return
	}

	c.Set(&quot;JWT_PAYLOAD&quot;, claims)
	identity := mw.IdentityHandler(c)

	if identity != nil {
		c.Set(mw.IdentityKey, identity)
	}

	if !mw.Authorizator(identity, c) {
		mw.unauthorized(c, http.StatusForbidden, mw.HTTPStatusMessageFunc(ErrForbidden, c))
		return
	}

	c.Next()
}
</code></pre><p>分析上面的代码，我们可以知道，middlewareImpl的Bearer认证流程为：</p><p><strong>第一步</strong>：调用<code>GetClaimsFromJWT</code>函数，从HTTP请求中获取Authorization Header，并解析出Token字符串，进行认证，最后返回Token Payload。</p><p><strong>第二步</strong>：校验Payload中的<code>exp</code>是否超过当前时间，如果超过就说明Token过期，校验不通过。</p><p><strong>第三步</strong>：给gin.Context中添加<code>JWT_PAYLOAD</code>键，供后续程序使用（当然也可能用不到）。</p><p><strong>第四步</strong>：通过以下代码，在gin.Context中添加IdentityKey键，IdentityKey键可以在创建<code>GinJWTMiddleware</code>结构体时指定，这里我们设置为<code>middleware.UsernameKey</code>，也就是username。</p><pre><code>identity := mw.IdentityHandler(c)

if identity != nil {
    c.Set(mw.IdentityKey, identity)
}
</code></pre><p>IdentityKey键的值由IdentityHandler函数返回，IdentityHandler函数为：</p><pre><code>func(c *gin.Context) interface{} {
    claims := jwt.ExtractClaims(c)

    return claims[jwt.IdentityKey]
}
</code></pre><p>上述函数会从Token的Payload中获取identity域的值，identity域的值是在签发Token时指定的，它的值其实是用户名，你可以查看<a href="https://github.com/marmotedu/iam/blob/v1.0.0/internal/apiserver/auth.go#L177">payloadFunc</a>函数了解。</p><p><strong>第五步</strong>：接下来，会调用<code>Authorizator</code>方法，<code>Authorizator</code>是一个callback函数，成功时必须返回真，失败时必须返回假。<code>Authorizator</code>也是在创建GinJWTMiddleware时指定的，例如：</p><pre><code>func authorizator() func(data interface{}, c *gin.Context) bool {    
    return func(data interface{}, c *gin.Context) bool {    
        if v, ok := data.(string); ok {    
            log.L(c).Infof(&quot;user `%s` is authenticated.&quot;, v)         
                                                                     
            return true                            
        }                                                        
                                                                 
        return false                     
    }    
}    
</code></pre><p><code>authorizator</code>函数返回了一个匿名函数，匿名函数在认证成功后，会打印一条认证成功日志。</p><h2>IAM项目认证功能设计技巧</h2><p>我在设计IAM项目的认证功能时，也运用了一些技巧，这里分享给你。</p><h3>技巧1：面向接口编程</h3><p>在使用<a href="https://github.com/marmotedu/iam/blob/v1.0.0/internal/pkg/middleware/auth/auto.go#L30">NewAutoStrategy</a>函数创建auto认证策略时，传入了<a href="https://github.com/marmotedu/iam/blob/v1.0.0/internal/pkg/middleware/auth.go#L12">middleware.AuthStrategy</a>接口类型的参数，这意味着Basic认证和Bearer认证都可以有不同的实现，这样后期可以根据需要扩展新的认证方式。</p><h3>技巧2：使用抽象工厂模式</h3><p><a href="https://github.com/marmotedu/iam/blob/v1.0.0/internal/apiserver/auth.go">auth.go</a>文件中，通过newBasicAuth、newJWTAuth、newAutoAuth创建认证策略时，返回的都是接口。通过返回接口，可以在不公开内部实现的情况下，让调用者使用你提供的各种认证功能。</p><h3>技巧3：使用策略模式</h3><p>在auto认证策略中，我们会根据HTTP 请求头<code>Authorization: XXX X.Y.X</code>中的XXX来选择并设置认证策略（Basic 或 Bearer）。具体可以查看<code>AutoStrategy</code>的<a href="https://github.com/marmotedu/iam/blob/v1.0.0/internal/pkg/middleware/auth/auto.go#L38">AuthFunc</a>函数：</p><pre><code>func (a AutoStrategy) AuthFunc() gin.HandlerFunc {
	return func(c *gin.Context) {
		operator := middleware.AuthOperator{}
		authHeader := strings.SplitN(c.Request.Header.Get(&quot;Authorization&quot;), &quot; &quot;, 2)
        ...
		switch authHeader[0] {
		case &quot;Basic&quot;:
			operator.SetStrategy(a.basic)
		case &quot;Bearer&quot;:
			operator.SetStrategy(a.jwt)
			// a.JWT.MiddlewareFunc()(c)
		default:
			core.WriteResponse(c, errors.WithCode(code.ErrSignatureInvalid, &quot;unrecognized Authorization header.&quot;), nil)
			c.Abort()

			return
		}

		operator.AuthFunc()(c)

		c.Next()
	}
}
</code></pre><p>上述代码中，如果是Basic，则设置为Basic认证方法<code>operator.SetStrategy(a.basic)</code>；如果是Bearer，则设置为Bearer认证方法<code>operator.SetStrategy(a.jwt)</code>。 <code>SetStrategy</code>方法的入参是AuthStrategy类型的接口，都实现了<code>AuthFunc() gin.HandlerFunc</code>函数，用来进行认证，所以最后我们调用<code>operator.AuthFunc()(c)</code>即可完成认证。</p><h2>总结</h2><p>在IAM项目中，iam-apiserver实现了Basic认证和Bearer认证，iam-authz-server实现了Bearer认证。这一讲重点介绍了iam-apiserver的认证实现。</p><p>用户要访问iam-apiserver，首先需要通过Basic认证，认证通过之后，会返回JWT Token和JWT Token的过期时间。前端将Token缓存在LocalStorage或Cookie中，后续的请求都通过Token来认证。</p><p>执行Basic认证时，iam-apiserver会从HTTP Authorization Header中解析出用户名和密码，将密码再加密，并和数据库中保存的值进行对比。如果不匹配，则认证失败，否则认证成功。认证成功之后，会返回Token，并在Token的Payload部分设置用户名，Key为 username 。</p><p>执行Bearer认证时，iam-apiserver会从JWT Token中解析出Header和Payload，并从Header中获取加密算法。接着，用获取到的加密算法和从配置文件中获取到的密钥对Header.Payload进行再加密，得到Signature，并对比两次的Signature是否相等。如果不相等，则返回 HTTP 401 Unauthorized 错误；如果相等，接下来会判断Token是否过期，如果过期则返回认证不通过，否则认证通过。认证通过之后，会将Payload中的username添加到gin.Context类型的变量中，供后面的业务逻辑使用。</p><p>我绘制了整个流程的示意图，你可以对照着再回顾一遍。</p><p><img src="https://static001.geekbang.org/resource/image/64/7e/642a010388e759dd76d411055bbd637e.jpg?wh=2248x1104" alt=""></p><h2>课后练习</h2><ol>
<li>走读<code>github.com/appleboy/gin-jwt</code>包的<code>GinJWTMiddleware</code>结构体的<a href="https://github.com/appleboy/gin-jwt/blob/v2.6.4/auth_jwt.go#L407">GetClaimsFromJWT</a>方法，分析一下：GetClaimsFromJWT方法是如何从gin.Context中解析出Token，并进行认证的？</li>
<li>思考下，iam-apiserver和iam-authzserver是否可以使用同一个认证策略？如果可以，又该如何实现？</li>
</ol><p>欢迎你在留言区与我交流讨论，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">本文的意思是说正常的生产环境下，iam-apiserver和iam-authz-server的api的认证功能其实都应该放到网关来实现的，本文之所以由iam项目亲自来实现就是为了方便讲解认证的具体实现方法，我理解的对不对？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 老哥理解的没毛病</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-26 00:25:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7a/d2/4ba67c0c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sch0ng</span>
  </div>
  <div class="_2_QraFYR_0">服务端实现Basic和Bearer认证的详细方案。<br>配合源码和架构图理解。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-14 22:04:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIW5xLKMIwlibBXdP5sGVqhXAGuLYk7XFBrhzkFytlKicjNpSHIKXQclDUlSbD9s2HDuOiaBXslCqVbg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f23c82</span>
  </div>
  <div class="_2_QraFYR_0">麻烦问下authserver什么时候派发的jwt token？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: iamctl jwt sign 通过这个命令签发token，authzserver解析token进行认证</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-29 10:55:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/51/84/5b7d4d95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冷峰</span>
  </div>
  <div class="_2_QraFYR_0">为什么每个用户都要有一个SecretKey， 所有的用户用同一个SecretKey不行吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这样不安全了，A、B、C都使用同一个secretKey，那都依赖于同一个secretKey，如果A想更改secretKey的过期时间，不就影响到B、C了。secretKey也属于用户资源，每个用于都应该有一个，符合产品设计思路</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-17 00:01:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/23/df/367f2c75.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🌀🐑hfy🐣</span>
  </div>
  <div class="_2_QraFYR_0">请问老师为什么bearer认证里面还要basic认证？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: bearer认证中没有basic认证。是不是理解错了？可以再看看，有问题再留言</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-26 17:40:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5b/66/ad35bc68.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>党</span>
  </div>
  <div class="_2_QraFYR_0">jwt需要后端解析并从缓存中拿用户对应秘钥在进行运算进行鉴权，这些流程是不是有点复杂和多余啊，登录时候直接随机生成一个token（uuid hash）传给前端并保存到缓存中，缓存中token直接对应用户的session，每次前端传过来token 根据是否能用token获取缓存中的session来鉴权 这样岂不是实现简单 也安全啊  </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这样做不太合理：<br>1. 如果很多用户，那应用程序会缓存并维护很多session，对应用程序是一种压力，如果应用程序重启，那登录会失效，这种是不合理的<br>2. jwt token中能够包含很多信息，可以基于这些信息做更多的认证逻辑</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-11 19:39:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLDUJyeq54fiaXAgF62tNeocO3lHsKT4mygEcNoZLnibg6ONKicMgCgUHSfgW8hrMUXlwpNSzR8MHZwg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>types</span>
  </div>
  <div class="_2_QraFYR_0">根据文中所说<br> 秘钥对是给iam-authz-server使用的<br> 每个用户维护一个密钥<br>请问:<br>1. iam-authz-server jwt认证中的jwt token谁谁生成的？是客户端还是iam-auth-server<br>2. 如果是客户端生成的jwt token，说明客户端是需要有secret秘钥对的信息的，请问这样设计有什么优势？<br>跟通过用户名密码登陆后，由服务端生成jwt token这种方式相比较有什么优势<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. IAM项目写了一个工具来生成jwt token:<br><br>iamctl jwt sign &lt;secretID&gt; &lt;secretKey&gt;<br><br>2. 其实，这里有2个维度，需要解释。<br><br>第一个token vs password的优势对比：<br><br>token相比password更加安全 - token有过期时间，经过加密。<br>更丰富的信息 - 可以在token的payload中，携带更丰富的信息，实现更强大的认证功能。<br><br>第二个API通过密钥访问 VS 控制台通过密码访问。<br><br>通过第一个对比，显然Token比password更安全。<br><br>但如果是控制台，我们需要提供给用户一个友好的登陆界面，所以控制台一般通过密码登陆。登陆后生成临时token，后面继续走token认证。<br><br>当然，随着技术的发展，目前也有很多其它登陆方式比如：扫一扫、安全令牌等</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-05 01:08:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/2b/8e/4d24c872.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>season</span>
  </div>
  <div class="_2_QraFYR_0">第四步，设置HTTP Header username: colin 。<br><br>应该是 第四步，给gin.Context中添加 username: colin 。  ？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，是一样的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-13 20:38:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/2b/8e/4d24c872.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>season</span>
  </div>
  <div class="_2_QraFYR_0">ParseWithClaims怎么理解？<br>func (p *Parser) ParseWithClaims(tokenString string, claims Claims, keyFunc Keyfunc) (*Token, error) {}<br><br>使用Claims来解析，并返回 token？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用来解析Token，并将解析后的claims存放到传入的claims变量中</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-13 17:31:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/2b/8e/4d24c872.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>season</span>
  </div>
  <div class="_2_QraFYR_0">技巧2:使用抽象工厂模式<br>auth.go文件中，通过newBasicAuth、newJWTAuth、newAutoAuth创建认证策略时，返回的 都是接口。通过返回接口，可以在不公开内部实现的情况下，让调用者使用你提供的各种认证 功能。<br><br><br>1. 不公开内部实现的情况下，是指不公开哪个函数的内部实现？<br>2. 让调用者使用你提供的各种认证功能，指的是哪些方法？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 相对于调用者来说，不用公开newBasicAuth、newJWTAuth、newAutoAuth方法的实现。<br>2. basic认证、jwt认证、和basic+jwt自动选择的认证方式</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-13 15:58:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5b/66/ad35bc68.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>党</span>
  </div>
  <div class="_2_QraFYR_0">jwt貌似不可以实现实时踢人吧 一个账号登录了 在登录一次 让上次的token失效 这个jwt不可以吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个JWT是没这种功能的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-29 12:43:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/64/3882d90d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yandongxiao</span>
  </div>
  <div class="_2_QraFYR_0">总结：<br>IAM系统采用 Basic + bearer 两种认证方式。Basic 认证要求输入用户名和密码，返回 JWT Token；虽然客户端在访问 iam-apiserver 或者 iam-auth-server 时，在 bearer 认证中携带该 Token，服务端对该请求进行认证。<br>1. 服务端basic认证实现逻辑：通过 gin middleware 实现了签发 JWT 的功能。jwt.New 对象在实例化时，传递多个回调函数，比如 Authentiactor, LoginResponse 等。<br>2. 服务端bearer认证实现逻辑：在 gin 中以 middleware 的方式存在，借助 jwt package 完成认证。认证完成后，会在 Context 中保存Username，方便后面的handler使用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的好细！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-27 18:18:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/59/a2/b28b1ffb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>姚力晓</span>
  </div>
  <div class="_2_QraFYR_0">如果 Redis down 掉，或者出现网络抖动，老师说会在新一期的特别放送里专门介绍下， 这个内容没看到？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 文章已经写完了，打磨后马上放出来</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-28 16:05:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/55/05/72d9aa41.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>指尖”^^的童话</span>
  </div>
  <div class="_2_QraFYR_0">项目有点大，如果是一步步实现的就更好了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-04 23:45:45</div>
  </div>
</div>
</div>
</li>
</ul>