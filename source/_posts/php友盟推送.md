---
title: php友盟推送
date: 2017-03-20 12:00:00
tags: [友盟,php]
categories: [技术]
---
## 友盟推送

### 准备工作

友盟推送官方[服务端demo](https://www.umeng.com/codecenter.html)

项目背景，服务催促推送，订单消息推送

话不多说，开始

### 友盟app推送

- 友盟demo修改，改动的地方较少

``` bash
<?php
namespace umeng;

class Umeng{
	protected $appkey           = NULL; 
	protected $appMasterSecret     = NULL;
	protected $timestamp        = NULL;
	protected $validation_token = NULL;

	function __construct($arr) {
		
		require_once EXTEND_PATH.'umeng\android\AndroidBroadcast.php';
		require_once EXTEND_PATH.'umeng\android\AndroidFilecast.php';
		require_once EXTEND_PATH.'umeng\android\AndroidGroupcast.php';
		require_once EXTEND_PATH.'umeng\android\AndroidUnicast.php';
		require_once EXTEND_PATH.'umeng\android\AndroidCustomizedcast.php';
		require_once EXTEND_PATH.'umeng\ios\IOSBroadcast.php';
		require_once EXTEND_PATH.'umeng\ios\IOSFilecast.php';
		require_once EXTEND_PATH.'umeng\ios\IOSGroupcast.php';
		require_once EXTEND_PATH.'umeng\ios\IOSUnicast.php';
		require_once EXTEND_PATH.'umeng\ios\IOSCustomizedcast.php';
		
		//设备配置
		$this->appkey = empty($arr['appkey'])?'':$arr['appkey'];
		$this->appMasterSecret = empty($arr['appMasterSecret'])?'':$arr['appMasterSecret'];
		$this->timestamp = strval(time());
	}
	
	/**
	 * 广播(broadcast): 向安装该App的所有设备发送消息。
	 * $tiker     通知栏提示文字
	 * $title     通知标题
	 * $text      通知文字描述
	 * $after_open   点击"通知"的后续行为，"go_app": 打开应用
                                   "go_url": 跳转到URL
                                   "go_activity": 打开特定的activity
                                   "go_custom": 用户自定义内容。
       $after_val   选填        after_open 后续操作参数
	 */
	
	function sendAndroidBroadcast($ticker,$title,$text,$after_open,$after_val='') {
	    $result = array('stauts'=>false,'msg'=>'',);
		try {
			$brocast = new \AndroidBroadcast();
			$brocast->setAppMasterSecret($this->appMasterSecret);
			$brocast->setPredefinedKeyValue("appkey",  $this->appkey);
			$brocast->setPredefinedKeyValue("timestamp",$this->timestamp);
			$brocast->setPredefinedKeyValue("ticker",   $ticker);   // 必填 通知栏提示文字
			$brocast->setPredefinedKeyValue("title",    $title);    // 必填 通知标题
			$brocast->setPredefinedKeyValue("text",     $text);
			$brocast->setPredefinedKeyValue("after_open",$after_open);
			
			switch ($after_open){
				case 'go_url':
					$brocast->setPredefinedKeyValue("url",       $after_val);
					break;
				case 'go_activity':
					$brocast->setPredefinedKeyValue("activity",  $after_val);
					break;
			}
			// Set 'production_mode' to 'false' if it's a test device. 
			// For how to register a test device, please see the developer doc.
			$brocast->setPredefinedKeyValue("production_mode", "true");
			// [optional]Set extra fields
			//$brocast->setExtraField("test", "helloworld");
// 			print("Sending broadcast notification, please wait...\r\n");
			$result['msg'] = $brocast->send();
			$result['stauts'] = true;
            return $result;
		} catch (\Exception $e) {
		    $result['msg'] = $e->getMessage();
		    return $result;
		}
	}

	/**
	 * 单播(unicast): 向指定的设备发送消息，包括向单个device_token或者单个alias发消息。
	 * $device_token  设备唯一表示
	 * $tiker     通知栏提示文字
	 * $title     通知标题
	 * $text      通知文字描述
	 * $custom    自定义内容
	 * $after_open   点击"通知"的后续行为，"go_app": 打开应用
                                   "go_url": 跳转到URL
                                   "go_activity": 打开特定的activity
                                   "go_custom": 用户自定义内容。
       $after_val   选填        after_open 后续操作参数

	 */
	function sendAndroidUnicast($device_token,$text,$custom,$ticker='XXX',$title='XXX',$after_open='',$after_val='') {
	    $result = array('stauts'=>false,'msg'=>'',);
	    try {
			$unicast = new \AndroidUnicast();
			$unicast->setAppMasterSecret($this->appMasterSecret);
			$unicast->setPredefinedKeyValue("appkey",           $this->appkey);
			$unicast->setPredefinedKeyValue("timestamp",        $this->timestamp);
			// Set your device tokens here
			$unicast->setPredefinedKeyValue("device_tokens",    $device_token); 
			$unicast->setPredefinedKeyValue("ticker",           $ticker);
			$unicast->setPredefinedKeyValue("title",            $title);
			$unicast->setPredefinedKeyValue("text",             $text);
			$unicast->setPredefinedKeyValue("after_open",       $after_open);
			$unicast->setPredefinedKeyValue("custom",$custom);
			switch ($after_open){
				case 'go_url':
       			 	$unicast->setPredefinedKeyValue("url",       $after_val);
        			break;
			    case 'go_activity':
			        $unicast->setPredefinedKeyValue("activity",  $after_val);
			        break;			
			}
			// Set 'production_mode' to 'false' if it's a test device. 
			// For how to register a test device, please see the developer doc.
			$checkArr = $unicast->setPredefinedKeyValue("production_mode", "false");
	        $sendArr = json_decode($unicast->send(),true);
            if ($sendArr['ret']!='SUCCESS'){
                $result['msg'] = "推送失败，错误码".$sendArr['data']['error_code'];
            }else{
                $result['stauts'] = true;
                $result['msg']="推送成功";
            }
            return $result;
		} catch (\Exception $e) {
		    $result['msg'] = $e->getMessage();
		    return $result;
		}
	}

	/**
	 * IOS广播(broadcast): 向安装该App的所有设备发送消息。
	 * $alert  内容
	 */
	function sendIOSBroadcast($alert) {
	    $result = array('stauts'=>false,'msg'=>'',);
		try {
			$brocast = new \IOSBroadcast();
			$brocast->setAppMasterSecret($this->appMasterSecret);
			$brocast->setPredefinedKeyValue("appkey",           $this->appkey);
			$brocast->setPredefinedKeyValue("timestamp",        $this->timestamp);

			$brocast->setPredefinedKeyValue("alert", $alert);
			//$brocast->setPredefinedKeyValue("badge", 0);
			//$brocast->setPredefinedKeyValue("sound", "chime");
			// Set 'production_mode' to 'true' if your app is under production mode
			$brocast->setPredefinedKeyValue("production_mode", "true");
			// Set customized fields
			//$brocast->setCustomizedField("test", "helloworld");
			print("Sending broadcast notification, please wait...\r\n");
            $result['msg'] = $brocast->send();
			$result['stauts'] = true;
            return $result;
		} catch (\Exception $e) {
		    $result['msg'] = $e->getMessage();
		    return $result;
		}
	}

	/**
	 * 单播(unicast): 向指定的设备发送消息，包括向单个device_token或者单个alias发消息。
	 * $device_tokens   设备唯一表示
	 * $alert      内容
	 * $custom     自定义内容
	 */
	function sendIOSUnicast($device_tokens,$alert,$custom) {
	    $result = array('stauts'=>false,'msg'=>'',);
		try {
			$unicast = new \IOSUnicast();
			$unicast->setAppMasterSecret($this->appMasterSecret);
			$unicast->setPredefinedKeyValue("appkey",           $this->appkey);
			$unicast->setPredefinedKeyValue("timestamp",        $this->timestamp);
			// Set your device tokens here
			$unicast->setPredefinedKeyValue("device_tokens",    $device_tokens); 
			$unicast->setPredefinedKeyValue("alert", $alert);
			$unicast->setPredefinedKeyValue("badge", 0);
			$unicast->setPredefinedKeyValue("sound", "chime");
			
			// Set 'production_mode' to 'true' if your app is under production mode
			$unicast->setPredefinedKeyValue("production_mode", "false");
			
			// Set customized fields
			$unicast->setCustomizedField("custom",$custom);
			//$unicast->setCustomizedField("test", "helloworld");
			$sendArr = json_decode($unicast->send(),true);
            if ($sendArr['ret']!='SUCCESS'){
                $result['msg'] = "推送失败，错误码".$sendArr['data']['error_code'];
            }else{
                $result['stauts'] = true;
                $result['msg']="推送成功";
            }
            return $result;
		} catch (\Exception $e) {
		    $result['msg'] = $e->getMessage();
		    return $result;
		}
	}

}

```
业务需求，重点对单推作修改


#### 调用

- 点对点推送

``` bash
/**
* @desc: 点对点推送--支持两个以上的设备
* @param: $userIds 用户ID
* @param: $title 推送的标题
* @param: $custom 自定义参数
* @return boolear 是否成功
*/    
public function doPush($userIds,$title,$custom = array()){
	try {
		if (empty($title)){
			return array('code'=>'1001','msg'=>'推送信息标题不能为空');
		} 
        通过用户ID获取用的	device_token	
		/**
		* 推送设备device_tokens获取
		foreach ($device_tokens as $device_token){
                // 判断device_token  64位表示为苹果 否则为安卓
                if(strlen($device_token) == 64){
                    $data['ios_device_tokens'] .= $device_token.',';  
                }else {
                    $data['android_device_tokens'] .= $device_token.',';  
                }
            }
		rtrim($data['ios_device_tokens'] ,',');
		rtrim($data['android_device_tokens'] ,',');
		**/
		//Android设备 
		$config = array(
		  'appkey'=>config('umeng.androidKey'),  
		  'appMasterSecret'=>config('umeng.androidMasterSecret'),  
		);
		// 导入友盟
		$umeng = new Umeng($config);
		$result = $umeng->sendAndroidUnicast($DeviceTokens['android_device_tokens'],$title,$custom);
		//iOS设备 
		$config = array(
		  'appkey'=>config('umeng.iosKey'),  
		  'appMasterSecret'=>config('umeng.iosMasterSecret'),  
		);
		// 导入友盟
		$umeng = new Umeng($config);
		$resultIOS = $umeng->sendIOSUnicast($DeviceTokens['ios_device_tokens'],$title,$custom);
		进行下一步处理
	  } catch (Exception $e) {
		  return array('code'=>'1001','msg'=>$e->getMessage());
	}
}

```
就这么多，比较简单吧，后续业务复杂点在改进
