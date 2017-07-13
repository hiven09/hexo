---
title: php支付宝多账号支付
date: 2017-03-18 12:00:00
tags: [支付宝,php]
categories: [技术]
---
## 支付宝进行app支付

### 准备工作

支付宝支付的官方[服务端demo](https://docs.open.alipay.com/54/106370/)

项目背景，多个商家在同一个平台支付，根据商家号去查询商家的appid(支付宝分配给开发者的应用ID)
支付宝的应用公钥和私钥以及通过应用公钥生成的支付宝公钥可以多个商家公用一个(方便，给个赞)

商户生成签名字符串所使用的签名算法类型，目前支持RSA2和RSA，推荐使用RSA2

支付宝审核效率较快

话不多说，开始

### 支付宝支付请求

支付宝支付都是json格式

#### 起调

- 支付宝支付起调

``` bash
public function alipay_app(){
	//构造业务请求参数的集合(订单信息)
	$content = array();
	$content['subject'] = $this->subject;//商品的标题/交易标题/订单标题/订单关键字等
	$content['out_trade_no'] = $this->out_trade_no;//商户网站唯一订单号
	$content['timeout_express'] = $this->timeout_express;//商品的标题/交易标题/订单标题/订单关键字等
	$content['total_amount'] = floatval($this->total_amount);//订单总金额(必须定义成浮点型)
	$content['product_code'] = $this->product_code;//销售产品码，商家和支付宝签约的产品码，为固定值QUICK_MSECURITY_PAY
	$con = json_encode($content);//$content是biz_content的值,将之转化成字符串
	//构造公共参数
	unset($parameter);
	$parameter = array(
		"app_id" => $this->app_id,//支付宝分配给开发者的应用ID
		"method" => $this->method,//接口名称
		"charset" => $this->charset,//请求使用的编码格式
		"sign_type" => $this->sign_type,//商户生成签名字符串所使用的签名算法类型
		"timestamp"  => date("Y-m-d H:i:s"),//发送请求的时间
		"version"	=> $this->version,//调用的接口版本，固定为：1.0
		"notify_url"  => $this->notify_url,//支付宝服务器主动通知地址
		"biz_content"  => $con,//业务请求参数的集合,长度不限,json格式
	);
	//生成签名
	$Client = new AopClient();
	$paramStr = $Client->getSignContent($parameter);//签名字符串
	$sign = $Client->alonersaSign($paramStr,$this->rsaPrivateKey,$this->sign_type);
	$parameter['sign'] = $sign;
	writeLog("| APP支付 | 参数 | ".json_encode($parameter), config('PAY_LOG'),'alipay');
	$str = $Client->getSignContentUrlencode($parameter);
	return $str;
}

```

#### 回调

- 支付宝支付回调

``` bash
public function notifyPay(){
	//验证生成签名
	$Client = new AopClient($this->alipayrsaPublicKey);
	$verify_result = $Client->rsaCheckV1($_POST,null);
	writeLog("| 支付回调 | 数据验证 | ".json_encode($verify_result),config('PAY_LOG'),'alipay');
	if($verify_result) {//验证成功
		/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
		//请在这里加上商户的业务逻辑程序代
		//——请根据您的业务逻辑来编写程序（以下代码仅作参考）——
		//获取支付宝的通知返回参数，可参考技术文档中服务器异步通知参数列表
		//商户订单号
		$out_trade_no = $_POST['out_trade_no'];
		//支付宝交易号
		$trade_no = $_POST['trade_no'];
		//交易状态
		$trade_status = $_POST['trade_status'];
		if($_POST['trade_status'] == 'TRADE_FINISHED') {
			return 'success';
			//判断该笔订单是否在商户网站中已经做过处理
			//如果没有做过处理，根据订单号（out_trade_no）在商户网站的订单系统中查到该笔订单的详细，并执行商户的业务程序
			//请务必判断请求时的total_fee、seller_id与通知时获取的total_fee、seller_id为一致的
			//如果有做过处理，不执行商户的业务程序
			//注意：
			//退款日期超过可退款期限后（如三个月可退款），支付宝系统发送该交易状态通知
			//调试用，写文本函数记录程序运行情况是否正常
			//logResult("这里写入想要调试的代码变量值，或其他运行的结果记录");
		}
		else if ($_POST['trade_status'] == 'TRADE_SUCCESS') {
			return 'success';
			//判断该笔订单是否在商户网站中已经做过处理
			//如果没有做过处理，根据订单号（out_trade_no）在商户网站的订单系统中查到该笔订单的详细，并执行商户的业务程序
			//请务必判断请求时的total_fee、seller_id与通知时获取的total_fee、seller_id为一致的
			//如果有做过处理，不执行商户的业务程序
			//注意：
			//付款完成后，支付宝系统发送该交易状态通知
			//调试用，写文本函数记录程序运行情况是否正常
			//logResult("这里写入想要调试的代码变量值，或其他运行的结果记录");
		}
		else if ($_POST['trade_status'] == 'TRADE_CLOSED') {
			return 'success';
			//判断该笔订单是否在商户网站中已经做过处理
			//如果没有做过处理，根据订单号（out_trade_no）在商户网站的订单系统中查到该笔订单的详细，并执行商户的业务程序
			//请务必判断请求时的total_fee、seller_id与通知时获取的total_fee、seller_id为一致的
			//如果有做过处理，不执行商户的业务程序
			//注意：
			//付款完成后，支付宝系统发送该交易状态通知
			//调试用，写文本函数记录程序运行情况是否正常
			//logResult("这里写入想要调试的代码变量值，或其他运行的结果记录");
		}
		//——请根据您的业务逻辑来编写程序（以上代码仅作参考）——
		return 'fail';
		//echo "success";	 //请不要修改或删除
		/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
	}
	else {
		return 'fail';
		//验证失败
		// echo "fail";
		//调试用，写文本函数记录程序运行情况是否正常
		//logResult("这里写入想要调试的代码变量值，或其他运行的结果记录");
	}
}

```

#### 退款

- 支付宝支付退款

``` bash
public function refund($param){
	$parameter = $result = [];
	try {
		$aop = new AopClient();
		$aop->gatewayUrl = 'https://openapi.alipay.com/gateway.do';
		$aop->appId = $this->app_id;
		$aop->rsaPrivateKey = $this->rsaPrivateKey;
		$aop->alipayrsaPublicKey = $this->alipayrsaPublicKey;
		$aop->apiVersion = '1.0';
		$aop->signType = 'RSA';
		$aop->postCharset='UTF-8';
		$aop->format='json';
		$parameter = [
			"trade_no" => $param['trade_no'],
			"out_trade_no" => $param['out_trade_no'],
			"out_request_no" => $param['out_request_no'],
			"refund_reason" => '正常退款',
			"refund_amount"=>$param['refund_money'],
		];
		$request = new request\AlipayTradeRefundRequest ();
		$request->setBizContent(json_encode($parameter));
		$result = $aop->execute ($request);
		$responseNode = 'alipay_trade_refund_response';//str_replace(".", "_", $request->getApiMethodName()) . "_response";
		$data = $result->$responseNode;
		$resultCode = $data->code;
		if(empty($resultCode) || $resultCode != 10000){
			$m = $data->sub_msg?$data->sub_msg:$data->msg;
			throw new \Exception($m, 1);
		}
		$ret['status'] = 'success';
		$ret['msg'] = '退款成功';
		$ret['data'] = $data;
	} catch (\Exception $e) {
		$ret['status'] = 'fail';
		$ret['msg'] = $e->getMessage();
	}
	writeLog("| 支付宝退款 | {$ret['status']}-{$ret['msg']} | 参数:".json_encode($parameter)." | 结果:".json_encode($result), config('PAY_LOG'),'refund');
	return $ret;
}
``` 
支付宝支付相对比较简单，有什么问题可以联系我喔！
