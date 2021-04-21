# gochat

[![golang](https://img.shields.io/badge/Language-Go-green.svg?style=flat)](https://golang.org)
[![GitHub release](https://img.shields.io/github/release/shenghui0779/gochat.svg)](https://github.com/shenghui0779/gochat/releases/latest)
[![pkg.go.dev](https://img.shields.io/badge/dev-reference-007d9c?logo=go&logoColor=white&style=flat)](https://pkg.go.dev/github.com/shenghui0779/gochat)
[![MIT license](http://img.shields.io/badge/license-MIT-brightgreen.svg)](http://opensource.org/licenses/MIT)

微信 Go SDK（支付、公众号、小程序）

| 目录  | 对应                         | 功能                                             |
| ---- | ---------------------------- | ----------------------------------------------- |
| /mch | 微信支付（普通商户直连模式）      | 下单、支付、退款、查询、委托代扣、企业付款、企业红包 等  |
| /oa  | 微信公众号（Official Accounts）| 网页授权、用户管理、模板消息、菜单管理、客服、事件消息 等 |
| /mp  | 微信小程序（Mini Program）     | 小程序授权、数据解密、二维码、消息发送、事件消息 等      |

## 获取

```sh
go get -u github.com/shenghui0779/gochat
```

## 文档

- [API Reference](https://pkg.go.dev/github.com/shenghui0779/gochat)
- [支付](https://github.com/shenghui0779/gochat/wiki/支付)
- [公众号](https://github.com/shenghui0779/gochat/wiki/公众号)
- [小程序](https://github.com/shenghui0779/gochat/wiki/小程序)




## 说明

- 支持 Go1.11+
- 注意：因 `access_token` 小程序与公众号的每日获取次数有限且含有效期，故服务端应妥善保存 `access_token` 并定时刷新
- 配合 [yiigo](https://github.com/shenghui0779/yiigo) 使用，可以更方便的操作 `MySQL`、`MongoDB` 与 `Redis` 等

**Enjoy 😊**


## 补充

#### 微信支付回调处理这个项目没有支持所以要手动处理

如
```golang
package wxpay

import (
	"crypto/md5"
	"encoding/xml"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"reflect"
	"sort"
	"strings"

	"github.com/golang/glog"
)

//微信支付回调

const (
	AckSuccess = `<xml><return_code><![CDATA[SUCCESS]]></return_code></xml>`
	AckFail    = `<xml><return_code><![CDATA[FAIL]]></return_code></xml>`
)

//微信 商户Key
var WXPApiKey string = ""

type WXPayNotify struct {
	ReturnCode    string `xml:"return_code"`
	ReturnMsg     string `xml:"return_msg"`
	Appid         string `xml:"appid"`
	MchID         string `xml:"mch_id"`
	DeviceInfo    string `xml:"device_info"`
	NonceStr      string `xml:"nonce_str"`
	Sign          string `xml:"sign"`
	ResultCode    string `xml:"result_code"`
	ErrCode       string `xml:"err_code"`
	ErrCodeDes    string `xml:"err_code_des"`
	Openid        string `xml:"openid"`
	IsSubscribe   string `xml:"is_subscribe"`
	TradeType     string `xml:"trade_type"`
	BankType      string `xml:"bank_type"`
	TotalFee      int64  `xml:"total_fee"`
	FeeType       string `xml:"fee_type"`
	CashFee       int64  `xml:"cash_fee"`
	CashFeeType   string `xml:"cash_fee_type"`
	CouponFee     int64  `xml:"coupon_fee"`
	CouponCount   int64  `xml:"coupon_count"`
	CouponID0     string `xml:"coupon_id_0"`
	CouponFee0    int64  `xml:"coupon_fee_0"`
	TransactionID string `xml:"transaction_id"`
	OutTradeNo    string `xml:"out_trade_no"`
	Attach        string `xml:"attach"`
	TimeEnd       string `xml:"time_end"`
}

/*
* 微信回调入口
* param w http.ResponseWriter
* param req *http.Request
 */
func HandleWXNotify(w http.ResponseWriter, req *http.Request) {
	log.Println("调用支付")
	glog.V(8).Info(req)

	if req.Method != "POST" {
		fmt.Fprint(w, AckFail)
		return
	}

	bodydata, err := ioutil.ReadAll(req.Body)
	if err != nil {
		glog.Error(err)
		fmt.Fprint(w, AckFail)
		return
	}

	glog.Info(string(bodydata))
	var wxn WXPayNotify
	err = xml.Unmarshal(bodydata, &wxn)
	if err != nil {
		glog.Error(err)
		fmt.Fprint(w, AckFail)
		return
	}
	if ProcessWX(wxn) {
		glog.Info("PROCESSWX SUCCESS")
		fmt.Fprint(w, AckSuccess)
		return
	}

	fmt.Fprint(w, AckFail)
}

/*
* 处理微信订单
* param wxn WXPayNotify
* reply true/false
 */
func ProcessWX(wxn WXPayNotify) bool {

	if !WXPayVerify(wxn) {
		glog.Warning("SIGN FAILED")
		return false
	}

	if !(wxn.ReturnCode == "SUCCESS" && wxn.ResultCode == "SUCCESS") {
		glog.Warning("INVALID STATUS", wxn)
		return false
	}

	/*
		业务逻辑 start

		.
		.
		.

		业务逻辑 end
	*/
	return true
}

/*
* 支付信息验证
* param data WXPayNotify
* reply true/false
 */
func WXPayVerify(data WXPayNotify) bool {
	glog.Info(data)
	sign := WXmd5Sign(data)
	if data.Sign == sign {
		return true
	} else {
		glog.V(8).Info(data.Sign, "  !=  ", sign)
		glog.Warning("WEIXIN PAY VERIFY FAIL")
	}
	return false
}

/*
* md5签名
* param data interface{}
* reply sign 签名字符串
 */
func WXmd5Sign(data interface{}) (sign string) {
	val := make(map[string]string)
	datavalue := reflect.ValueOf(data)
	if datavalue.Kind() != reflect.Struct {
		glog.Warning("NOT A STRUCT ", data)
		return ""
	}
	var keys []string
	for i := 0; i < datavalue.NumField(); i++ {
		k := datavalue.Type().Field(i)
		kl := k.Tag.Get("xml")
		v := fmt.Sprintf("%v", datavalue.Field(i))

		if v != "" && v != "0" && kl != "sign" {
			val[kl] = v
			keys = append(keys, kl)
		}
	}
	sort.Strings(keys)
	var stra string
	for _, v := range keys {
		stra = stra + v + "=" + val[v] + "&"
	}
	strb := stra + "key=" + WXPApiKey
	glog.V(8).Info("SIGN STRING ", strb)
	hstr := md5.Sum([]byte(strb))

	sum := fmt.Sprintf("%x", hstr)
	sign = strings.ToUpper(sum)
	return sign
}

package main

import (
	"flag"
	"log"
	"net/http"
	"test/wxpay"

	"github.com/golang/glog"
)

func main() {
	flag.Parse()
	defer glog.Flush()

	mux := http.NewServeMux()
	mux.HandleFunc("/", wxpay.HandleWXNotify)

	log.Println("server is listening at", ":12138")
	log.Fatalln(http.ListenAndServe(":12138", mux))
}
```



