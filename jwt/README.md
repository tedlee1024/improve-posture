# JWT

`jwt`看[官方文档](https://jwt.io/introduction/) 就够了。下面用`Go`实践一下:

## 生成token

实际使用jwt时，一般会用[jwt-go](github.com/dgrijalva/jwt-go)这个第三方库。
这里试着手动签发token.

```Go

package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"fmt"
	"strings"
	"time"

	"github.com/dgrijalva/jwt-go"
)

type Header struct {
	Alg string `json:"alg"`
	Typ string `json:"typ"`
}

// 自定义的payload, 可以嵌入标准Claims
type Payload struct {
	jwt.StandardClaims
	Sub  string `json:"sub"`
	User string `json:"user"`
}

// 服务器保管，不可泄露
const Secret = "jwt_test"

func main() {
	token := GenToken()
	fmt.Println(token)
}

func GenToken() string {
	h := &Header{
		Alg: "HS256",
		Typ: "JWT",
	}
	p := &Payload{
		StandardClaims:jwt.StandardClaims{
			Issuer: "lfn",
			ExpiresAt: time.Now().Add(3 * time.Hour).Unix(),
	},
		Sub:  "test",
		User: "lfn",
	}

	hJson, _ := json.Marshal(h)
	pJson, _ := json.Marshal(p)
	hBase64URL := base64.URLEncoding.EncodeToString(hJson)
	pBase64URL := base64.URLEncoding.EncodeToString(pJson)

	// 计算hash之前剪除padding字符
	data := strings.TrimRight(hBase64URL, "=") + "." + strings.TrimRight(pBase64URL, "=")

	// 计算签名,这个辅助函数在下一节给出
	sig := SigBase64URLEncoded(data)

	token := data + "." + sig
	fmt.Println(token)

	return token
}


```

需要注意的是，header和payload在参与计算签名之前，需要trim掉padding字符; 计算出签名之后，仍然需要对签名用Base64URL编码，并且需要祛除右侧的padding符"=".
> padding字符详情见 [bas464](../encoding/base64.md)

所以实际计算token的算法如下:

```Go

data := TrimRightPadding(base64UrlEncode(header)) + "." +
  TrimRightPadding(base64UrlEncode(payload))

ha := HMACSHA256(data, secret)

sig := TrimRightPadding(base64UrlEncode(ha))

token := data + "." + sig  
  

```


> 有些一站式登录的服务，可能好几个服务共用一个`jwt secret`, 这样一个服务签发的token,同样可以在其他服务通过验证。


## 解析token

下面手动验证以下token的有效性。实际生产中，在验证通过之后，还可以对`payload`中一些字段如`过期时间`进行额外验证，保证该token还未失效。以下代码片段演示签名验证：

```Go
func ValidToken(token string) bool {
	idx := strings.LastIndex(token, ".")
	data, sign := token[:idx], token[idx+1:]

	sigFromTokenData := SigBase64URLEncoded(data)

	fmt.Printf("\n%-5s:%30s\n", "want", sign)
	fmt.Printf("%-5s:%3s\n", "got", sigFromTokenData)

	return sigFromTokenData == sign
}

func SigBase64URLEncoded(data string) string {
	ha := hmac.New(sha256.New, []byte(Secret))
	// Write Data to it
	ha.Write([]byte(data))
	sha := ha.Sum(nil)

	// base64URL编码
	rawSig := base64.URLEncoding.EncodeToString(sha)

	return strings.TrimRight(rawSig, "=")
}

```