---
title: php微信多账号支付
date: 2017-03-16 12:00:00
tags: [微信,php]
categories: [技术]
---
## wechat进行app支付

### 准备工作

微信支付的官方[服务端demo](https://pay.weixin.qq.com/wiki/doc/api/jsapi.php?chapter=11_1)
微信的demo藏的比较深

项目背景，多个商家在同一个平台支付，每个商家都要一个支付的appid(开发者的应用ID)和mch_id(微信支付分配的商户号)
商家微信公众平台需要上传当前app的信息(绑定支付的APPID)，审核成功后获取mch_id和key(商户支付密钥,参考开户邮件设置,必须配置,登录商户平台自行设置)
app_secret公众帐号secert(仅JSAPI支付的时候需要配置， 登录公众平台，进入开发者中心可设置)

吐槽下微信的审核周期比较长

话不多说，开始

### 微信支付请求

微信支付都是xml格式

#### 起调

- 微信支付起调

``` bash
    public function sendRequest() {
        $inputObj = new \WxPayUnifiedOrder();
        //构造业务请求参数的集合(订单信息)
        $key = $this->key;//商户支付密钥
        $inputObj->SetAppid($this->appid);
        $inputObj->SetMch_id($this->mch_id);
        $inputObj->SetBody($this->body);
        $inputObj->SetOut_trade_no($this->out_trade_no);
        $inputObj->SetTotal_fee($this->total_fee);
        $inputObj->SetSpbill_create_ip($this->spbill_create_ip);
        $inputObj->SetNotify_url($this->notify_url);
        $inputObj->SetTrade_type($this->trade_type);

        $api = new \WxDoPay();
		//其中有两次签名
        $result = $api->unifiedOrder($inputObj,$key);    //获取要发送的数据
        if (!empty($result)&&!empty($result['prepay_id'])){
            $singObj = new  \WxPayJsApiPay();
            $singObj->SetAppid($result['appid']);
            $singObj->SetPartnerid($result['mch_id']);
            $singObj->SetPrepayid($result['prepay_id']);
            $singObj->SetPackage('Sign=WXPay');
            $singObj->SetNoncestr($result['nonce_str']);
            $singObj->SetTimestamp(time());
			//再次签名
            $singObj->SetSign($key);
            $result = $singObj->GetValues();
        }
		//写日志
        writeLog("| APP支付 | 参数 | ".json_encode($result), config('PAY_LOG'),'wxpay');
        return json_encode($result);
    }

```
- 遇到的坑
	
细心点：
微信统一支付接口调用有两次签名
再起调时
要进行三次签名
其中签名字段要按照说明文档来
起调参数都是小写很容易出错


#### 回调

- 微信支付回调

``` bash
    public function notifyPay($arr=''){   
		//如果返回成功则验证签名
		require_once 'lib/WxPay.Notify.php';
		//获取通知的数据
		$arr['key'] = empty($arr['key'])?$this->key:$arr['key'];
		$arr['appid'] = empty($arr['appid'])?$this->appid:$arr['appid'];
		$arr['mch_id'] = empty($arr['mch_id'])?$this->mch_id:$arr['mch_id'];
		$notify = new \WxPayNotify();
		return $notify->Handle($arr,false);
	}

```
- 遇到的坑
	
回调时 用:file_get_contents('php://input')
而不用 ：$GLOBALS['HTTP_RAW_POST_DATA']

#### 退款

- 微信支付退款

``` bash
    public function refund($pay_info,$data){
		$inputObj = new \WxPayRefund();
		// //构造业务请求参数的集合(订单信息)
		$inputObj->SetAppid($pay_info['app_id']);
		$inputObj->SetMch_id($pay_info['mch_id']);
		$inputObj->SetTransaction_id($pay_info['trade_no']);
		$inputObj->SetOut_refund_no($data['out_request_no']);
		$inputObj->SetTotal_fee($pay_info['money']*100);
		$inputObj->SetRefund_fee($data['refund_money']*100);
		$inputObj->SetOp_user_id($pay_info['mch_id']);
		$api = new \WxDoPay();
		$result = $api->refund($inputObj,$pay_info['cert_file'],$pay_info['key_file'],$pay_info['key'],6);  //获取要发送的数据
		writeLog("| 微信退款 | 参数:".json_encode($data)." | 返回值:".json_encode($result), config('PAY_LOG'),'refund');
		return $result;
	}

```

微信支付中，金额都是以分为单位，注意转化下就可以了


### 微信官方demo修改

demo改动的地方,主要是将参数改成自己引入，重点是key


- WxDoPay
``` bash
/**
 * 
 * 统一下单，WxPayUnifiedOrder中out_trade_no、body、total_fee、trade_type必填
 * appid、mchid、spbill_create_ip、nonce_str不需要填入
 * @param WxPayUnifiedOrder $inputObj
 * @param int $timeOut
 * @throws WxPayException
 * @return 成功时返回，其他抛异常
 */
public static function unifiedOrder($inputObj,$key, $timeOut = 60)
{
	$url = "https://api.mch.weixin.qq.com/pay/unifiedorder";
	//检测必填参数
	if(!$inputObj->IsOut_trade_noSet()) {
		throw new WxPayException("缺少统一支付接口必填参数out_trade_no！");
	}else if(!$inputObj->IsBodySet()){
		throw new WxPayException("缺少统一支付接口必填参数body！");
	}else if(!$inputObj->IsTotal_feeSet()) {
		throw new WxPayException("缺少统一支付接口必填参数total_fee！");
	}else if(!$inputObj->IsTrade_typeSet()) {
		throw new WxPayException("缺少统一支付接口必填参数trade_type！");
	}
	if (empty($key)){
		throw new WxPayException("缺少统一支付接口必填参数key！");
	}
	//关联参数
	if($inputObj->GetTrade_type() == "JSAPI" && !$inputObj->IsOpenidSet()){
		throw new WxPayException("统一支付接口中，缺少必填参数openid！trade_type为JSAPI时，openid为必填参数！");
	}
	if($inputObj->GetTrade_type() == "NATIVE" && !$inputObj->IsProduct_idSet()){
		throw new WxPayException("统一支付接口中，缺少必填参数product_id！trade_type为JSAPI时，product_id为必填参数！");
	}
	
	//异步通知url未设置，则使用配置文件中的url
	if(!$inputObj->IsNotify_urlSet()){
		throw new WxPayException("缺少统一支付接口必填异步通知url！");
// 			$inputObj->SetNotify_url(WxPayConfig::NOTIFY_URL);//异步通知url
	}
	//商户信息---修改by(hiven)
	if(!$inputObj->IsAppidSet()){
		throw new WxPayException("缺少统一支付接口必填公众账号appid！");
	}
	if(!$inputObj->IsMch_idSet()){
		throw new WxPayException("缺少统一支付接口必填商户号mch_id！");
	}
	if(!$inputObj->IsSpbill_create_ipSet()){
		throw new WxPayException("缺少统一支付接口必填终端ip！");
	} 
	//$inputObj->SetSpbill_create_ip("1.1.1.1");  	    
	$inputObj->SetNonce_str(self::getNonceStr());//随机字符串
	
	//签名
	$inputObj->SetSign($key);
	$xml = $inputObj->ToXml();
	$startTimeStamp = self::getMillisecond();//请求开始时间
	$response = self::postXmlCurl($xml, $url, false, $timeOut);
	$result = WxPayResults::Init($response,$key);
	self::reportCostTime($url, $startTimeStamp, $result);//上报请求花费时间
	
	return $result;
}
	
/**
 * 
 * 查询订单，WxPayOrderQuery中out_trade_no、transaction_id至少填一个
 * appid、mchid、spbill_create_ip、nonce_str不需要填入
 * @param WxPayOrderQuery $inputObj
 * @param int $timeOut
 * @throws WxPayException
 * @return 成功时返回，其他抛异常
 */
public static function orderQuery($inputObj,$key='', $timeOut = 6)
{
	$url = "https://api.mch.weixin.qq.com/pay/orderquery";
	//检测必填参数
	if(!$inputObj->IsOut_trade_noSet() && !$inputObj->IsTransaction_idSet()) {
		throw new WxPayException("订单查询接口中，out_trade_no、transaction_id至少填一个！");
	}
	if (!$inputObj->IsAppidSet()||!$inputObj->IsMch_idSet()){
		throw new WxPayException("订单查询接口中，app_id、mch_id必须存在！");
	}
	$inputObj->SetNonce_str(self::getNonceStr());//随机字符串
	
	$inputObj->SetSign($key);//签名
	$xml = $inputObj->ToXml();
	$startTimeStamp = self::getMillisecond();//请求开始时间
	$response = self::postXmlCurl($xml, $url, false, $timeOut);
	$result = WxPayResults::Init($response,$key);
	self::reportCostTime($url, $startTimeStamp, $result);//上报请求花费时间
	
	return $result;
}
/**
 * 申请退款，WxPayRefund中out_trade_no、transaction_id至少填一个且
 * out_refund_no、total_fee、refund_fee、op_user_id为必填参数
 * appid、mchid、spbill_create_ip、nonce_str不需要填入
 * @param WxPayRefund $inputObj
 * @param int $timeOut
 * @throws WxPayException
 * @return 成功时返回，其他抛异常
 */
public static function refund($inputObj,$cert_pem,$key_pem,$key, $timeOut = 6)
{
	$url = "https://api.mch.weixin.qq.com/secapi/pay/refund";
	//检测必填参数
	if(!$inputObj->IsOut_trade_noSet() && !$inputObj->IsTransaction_idSet()) {
		throw new WxPayException("退款申请接口中，out_trade_no、transaction_id至少填一个！");
	}else if(!$inputObj->IsOut_refund_noSet()){
		throw new WxPayException("退款申请接口中，缺少必填参数out_refund_no！");
	}else if(!$inputObj->IsTotal_feeSet()){
		throw new WxPayException("退款申请接口中，缺少必填参数total_fee！");
	}else if(!$inputObj->IsRefund_feeSet()){
		throw new WxPayException("退款申请接口中，缺少必填参数refund_fee！");
	}else if(!$inputObj->IsOp_user_idSet()){
		throw new WxPayException("退款申请接口中，缺少必填参数op_user_id！");
	}
	if(!$inputObj->IsAppidSet()){
		throw new WxPayException("退款申请接口中，缺少必填参数app_id！");
	}
	if(!$inputObj->IsMch_idSet()){
		throw new WxPayException("退款申请接口中，缺少必填参数mch_id！");
	}
	$inputObj->SetNonce_str(self::getNonceStr());//随机字符串
	$inputObj->SetSign($key);//签名
	$xml = $inputObj->ToXml();
	$startTimeStamp = self::getMillisecond();//请求开始时间
	$response = self::postXmlCurl($xml, $url, true, $timeOut,$cert_pem,$key_pem);
	$result = WxPayResults::Init($response,$key);
	self::reportCostTime($url, $startTimeStamp, $result);//上报请求花费时间
	return $result;
}
```
- 生成签名
``` bash
	/**
	 * 生成签名
	 * @return 签名，本函数不覆盖sign成员变量，如要设置签名需要调用SetSign方法赋值
	 */
	public function MakeSign($key='')
	{
		//签名步骤一：按字典序排序参数
		ksort($this->values);
		$string = $this->ToUrlParams();
		//签名步骤二：在string后加入KEY
		$key = empty($key)?WxPayConfig::KEY:$key;
		$string = $string . "&key=".$key;
		//签名步骤三：MD5加密
		$string = md5($string);
		//签名步骤四：所有字符转为大写
		$result = strtoupper($string);
		return $result;
	}
```
demo改动不大，主要是参数引用问题，其他改动的没有参数就添加参数
