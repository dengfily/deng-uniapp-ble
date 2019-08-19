<template>
	<view class="content">
		<view class="text-area">
			<text>心跳 : {{tempValue}} 次/分</text>
		</view>
		<image class="logo" src="/static/logo.png"></image>
		<navigator url="/pages/blePage/blePage" navigateBack hover-class="navigator-hover">
			<button type="primary">蓝牙操作</button>
		</navigator>
	</view>
</template>

<script>
	var NfcAdapter;  
    var NdefRecord;  
    var NdefMessage;  
    var waiting  
    var readyRead = false;  
    var nfcAdapter,main , pendingIntent,intentFiltersArray,techListsArray, IntentFilter 
	
	export default {
		data() {
			return {
				title: 'Hello World',
				deviceId: "",
				serviceId: "",
				tempValue: '',
				devicesList: [{
					"device": "初始设备"
				}],		
				devicesListNEW: [{
					"device": "初始设备"
				}],	
				servicesList: [{
					"services": "初始服务"
				}],	
				characteristicsLists: [{
					"characteristics": "初始服务"
				}],					
				characteristicId: [{
					//支持写入操作的特征值
					writeId: "",
					//支持notify操作的特征值
					notifyId: ""
				}],			
														
                currentNFCInfo: [], //NFC 读取消息；  
                bannerShow: false,  
                remark:"",  
                message: "",  
                count:0,  
                timestampHide:"",  
                timestampShow:"",
				temp_intent:{},													
			}
		},
		onLoad() {
			// this.listenNFCStatus();
		},
        onShow: function(){  
            console.log('onShow - page Show');  
            this.timestampShow = (new Date()).getTime();  
			console.log('onShow - this.timestampShow : ', this.timestampShow);
            var TimeScale =  this.timestampShow - this.timestampHide;  
			console.log('onShow - TimeScale : ', TimeScale);
                        nfcAdapter.enableForegroundDispatch(main, pendingIntent, intentFiltersArray, techListsArray);  
            if(this.count++ == 0){  
                // this.listenNFCStatus()  
				console.log('onShow - this.count++ == 0');
                return false;  
            }else if(TimeScale > 400){  
				console.log('onShow - TimeScale > 400');
                return false;  
            }else{  
				console.log('onShow - this.readData()');
                this.readData();  
            }  
        },  
        onHide:function(){  
            console.log("onHide");  
            this.timestampHide = (new Date()).getTime();  
            console.log('onHide - this.timestampHide : ', this.timestampHide);  
        },		
		methods: {
			listenNFCStatus() {  
				console.log('listenNFCStatus()');
                try {  
                    console.log('listenNFCStatus - Init NFC...');  
                    main = plus.android.runtimeMainActivity();  
                    var Intent = plus.android.importClass('android.content.Intent');  
                    var Activity = plus.android.importClass('android.app.Activity');  
                    var PendingIntent = plus.android.importClass('android.app.PendingIntent');  
                    IntentFilter = plus.android.importClass('android.content.IntentFilter');  
                    NfcAdapter = plus.android.importClass('android.nfc.NfcAdapter');  
                    nfcAdapter = NfcAdapter.getDefaultAdapter(main);  
                    var intent = new Intent(main, main.getClass());  
                    intent.addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP);  
                    pendingIntent = PendingIntent.getActivity(main, 0, intent, 0);  
                    var ndef = new IntentFilter("android.nfc.action.TECH_DISCOVERED");  
                    ndef.addDataType("*/*");  
                    intentFiltersArray = [ndef];  
                    techListsArray = [  
                        ["android.nfc.tech.IsoDep"],  
                        ["android.nfc.tech.NfcA"],  
                        ["android.nfc.tech.NfcB"],  
                        ["android.nfc.tech.NfcF"],  
                        ["android.nfc.tech.Ndef"],  
                        ["android.nfc.tech.NfcV"],  
                        ["android.nfc.tech.NdefFormatable"],  
                        ["android.nfc.tech.MifareClassic"],  
                        ["android.nfc.tech.MifareUltralight"]  
                    ];  

                } catch (e) {  
                    console.error(e);  
                }  
            },  
            handle_nfc_data() {  

                NdefRecord = plus.android.importClass("android.nfc.NdefRecord");  
                NdefMessage = plus.android.importClass("android.nfc.NdefMessage");  
                // main = plus.android.runtimeMainActivity();  
                var intent = main.getIntent();  
				console.log('handle_nfc_data - typeof : ', typeof(intent));
				this.temp_intent = intent;
				console.log("handle_nfc_data - this.temp_intent:" + this.temp_intent);
                console.log("handle_nfc_data - action type:" + intent.getAction());  
                if ("android.nfc.action.TECH_DISCOVERED" == intent.getAction()) {  
					console.log("handle_nfc_data  - android.nfc.action.TECH_DISCOVERED");
                    if (readyRead) {  
						console.log("handle_nfc_data -readyRead - intent:" + intent);  
                        // this.handle_read(intent); 
						this.handle_read(this.temp_intent); 
                        readyRead = false;  
                    }  
                } else {  
                    // waiting.close();  
                    console.log("nfc读取失败")  
                }  
            },  
            handle_read(intent_params) {  
				console.log('handle_read - intent_params : ', intent_params);
                try {  
                    var content = "";  
                    waiting = plus.nativeUI.showWaiting("请将NFC标签靠近！");  
                    waiting.setTitle('请勿移开标签\n正在读取数据...');  
                    var tag = plus.android.importClass("android.nfc.Tag");  
                    waiting.close();  
                    var Parcelable = plus.android.importClass("android.os.Parcelable");  
                    // var rawmsgs = intent_params.getParcelableArrayExtra("android.nfc.extra.NDEF_MESSAGES");  
                    var rawmsgs = intent_params.getParcelableArrayExtra(NfcAdapter.EXTRA_NDEF_MESSAGES);  
					console.log('__read - rawmsgs : ', rawmsgs);
                    if (rawmsgs != null && rawmsgs.length > 0) {  
                        waiting.close();  
                        // console.log(JSON.stringify(rawmsgs[0]));  
                        var records = rawmsgs[0].getRecords();  
                        var result = records[0].getPayload();  
                        // console.log(result)  
                        if (result != null) {  
                            var s = plus.android.newObject("java.lang.String", result);  
                            console.log(s);  
                            this.currentNFCInfo = s;  
                        } else {  
                            this.currentNFCInfo = "";  
                        }  
                    } else {  
                        console.log("NFC获取失败a");  
                        uni.showToast({  
                            title: "NFC获取失败b.",  
                            icon: "none"  
                        });  
                    }  

                } catch (e) {  
                    console.log(e);  
                    console.log("NFC获取失败c,丢出异常");  
                    waiting.close();  
                    uni.showToast({  
                        title: "NFC获取失败d,丢出异常.",  
                        icon: "none"  
                    });  
                    //TODO handle the exception  
                }  
            },  
            readData() {  
                readyRead = true;  
            //  waiting = plus.nativeUI.showWaiting("请将NFC标签靠近！");  
                setTimeout(this.handle_nfc_data, 1000);  
            },	
		}
	}
</script>

<style>
	.content {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
	}

	.logo {
		height: 200upx;
		width: 200upx;
		margin-top: 200upx;
		margin-left: auto;
		margin-right: auto;
		margin-bottom: 50upx;
	}

	.text-area {
		display: flex;
		justify-content: center;
	}

	.title {
		font-size: 36upx;
		color: #8f8f94;
	}
</style>
