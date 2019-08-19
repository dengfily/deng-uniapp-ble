<template>
	<view class="ble-wrap">
		<view class="ble-act-btn">
			<button type="primary" @tap="toInitBLE" :class="isSearch ? 'noAllow' : ''">初始化</button>
			<button type="primary" @tap="closeBluetoothAdapter">关闭</button>
		</view>
		<view class="device-list" v-for="device in devicesList" :key="device">
			<view>设备名称:{{device.name}}</view>
			<view>设备id:{{device.deviceId}}</view>
			<button size="mini" type="primary" class="connect-btn" @tap="createBLEConnection(device)">{{device.connectStatus === 0 || device.connectStatus === 1 ? '连接' : '断开'}}</button>
		</view>
		<view>心跳:{{tempValue}}</view>
	</view>
</template>

<script>
	export default {
		data() {
			return {
				isSearch: '',
				isConncet: false,
				devicesList: [],
				deviceId: null,
				serviceId: [],
				characteristicsLists: [],
				tempValue: 0,
				characteristicId: [{
					writeId: '',
					notifyId: ''
				}]
			}
		},
		onHide() {
			this.closeBluetoothAdapter();
		},
		methods: {
			toInitBLE() {
				this.isSearch = true;
				//在页面加载时候初始化蓝牙适配器
				uni.openBluetoothAdapter({
					success: e => {
						console.log('初始化蓝牙成功:' + e.errMsg);
						console.log('uni.openBluetoothAdapter - e:' + JSON.stringify(e));
						//同时监听蓝牙连接状态
						this.onBLEConnectionStateChange();
						this.startBluetoothDeviceDiscovery(); // 开始搜索
					},
					fail: e => {
						console.log('初始化蓝牙失败，错误码：' + (e.errCode || e.errMsg));
						uni.showToast({
							title: '请打开蓝牙',
							icon: 'none'
						});
					}
				});
			},
			onBLEConnectionStateChange() {
				let count = 0;
				uni.onBLEConnectionStateChange(res => {
					// 该方法回调中可以用于处理连接意外断开等异常情况
					console.log(`蓝牙连接状态 -------------------------->`);
					console.log('uni.onBLEConnectionStateChange - res : ', JSON.stringify(res));
					if (!res.connected) {
						// if (this.isStop) return;
						console.log('断开低功耗蓝牙成功:');
						uni.showToast({
							icon: "none",
							title: "蓝牙已经断开",
							mask: false,
							duration: 3000
						});
						this.devicesList.map(item => {
							if (item.connectStatus !== 0) {
								item.connectStatus = 0;
							}
						})
						//在这里尝试重连
						//this.createBLEConnection();
						//关闭连接
						// this.closeBluetoothAdapter();
					}
				});
			},
			startBluetoothDeviceDiscovery() {
				//在页面显示的时候判断是都已经初始化完成蓝牙适配器若成功，则开始查找设备
				let self = this;
				console.log("开始搜寻智能设备");
				uni.showLoading({
					title: '正在搜索设备'
				});
				uni.startBluetoothDevicesDiscovery({
					//services: ['fff0'],
					success: res => {
						console.log("uni.startBluetoothDevicesDiscovery - res : " + JSON.stringify(res));
						this.onBluetoothDeviceFound();
						// 5s后停止搜索
						setTimeout(function() {
							self.stopBluetoothDevicesDiscovery();
						}, 5000);
					},
					fail: res => {
						console.log("查找设备失败!");
						uni.showToast({
							icon: "none",
							title: "查找设备失败！",
							duration: 3000
						})
					}
				});
			},
			onBluetoothDeviceFound() {
				console.log("监听寻找新设备");
				//this.getBluetoothDevices();
				uni.onBluetoothDeviceFound(devices => {
					console.log('开始监听寻找到新设备的事件');
					this.getBluetoothDevices();
				});
			},
			/**
			 * 获取在蓝牙模块生效期间所有已发现的蓝牙设备。包括已经和本机处于连接状态的设备。
			 */			
			getBluetoothDevices() {
				console.log("获取蓝牙设备");
				uni.getBluetoothDevices({
					success: res => {
						console.log('获取蓝牙设备成功:' + res.errMsg);
						console.log('uni.getBluetoothDevices - res : ', JSON.stringify(res));
						// this.stopBluetoothDevicesDiscovery();
						const deviceList = res.devices;
						for (let i = 0; i < deviceList.length; i++) {
							let eq = deviceList[i];
							if (eq.name.indexOf("Band") > -1) {
								// console.log("查找band设备：", eq);
								const list = this.devicesList.filter(item => item.deviceId === eq.deviceId);
								if (list.length === 0) {
									this.devicesList.push({...eq, connectStatus: 0});
								}
							}
						}
					},
					fail: e => {
						console.log('获取蓝牙设备错误，错误码：' + e.errCode);
					}
				});
			},
			/**
			 * 连接设备
			 */
			createBLEConnection(device) {
				if (device.connectStatus === 2) {
					this.closeBluetooth(device); // 断开连接
					return;
				}
				uni.showLoading({
					title: '正在连接设备',
					mask: true
				})
				device.connectStatus = 1; // 正在连接
				this.deviceId = device.deviceId; // 设备deviceId
				let self = this;
				console.log('设备蓝牙信息', device);
				uni.createBLEConnection({
					// 这里的 deviceId 需要已经通过 createBLEConnection 与对应设备建立链接
					deviceId: device.deviceId,
					success: res => {
						console.log('uni.createBLEConnection - res : ', JSON.stringify(res));
						console.log("设备连接成功！");
						device.connectStatus = 2; // 连接成功
						//延迟1.5s获取设备的services
						setTimeout(function() {
							this.isConncet = true;
							console.log("获取设备的services");
							self.getBLEDeviceServices();
						}, 1500);
					},
					fail: res => {
						console.log(JSON.stringify(res));
						console.log("设备连接失败！");
						this.isConncet = false;
						device.connectStatus = 0;
					}
				});
			},		
			/**
			 * 获取设备的服务ID
			 */
			getBLEDeviceServices() {
				let deviceId = this.deviceId;
				let serviceList = [];
				let self = this;
				uni.getBLEDeviceServices({
					// 这里的 deviceId 需要已经通过 createBLEConnection 与对应设备建立链接
					deviceId,
					success: res => {
						console.log('getBLEDeviceServices - res : ', JSON.stringify(res));
						serviceList = res.services;
						for (let i = 0; i < serviceList.length; i++) {
							let service = serviceList[i];
							console.log(JSON.stringify(service) + "----serviceID：" + service.uuid);
							//比对service是否是FFF0服务
							if (service.uuid.indexOf('0000180D-0000-1000-8000-00805F9B34FB') != -1) {
								self.serviceId = service.uuid;
								console.log("设备的serviceId： " + self.serviceId);
								//开始获取指定服务的特征值
								self.getBLEDeviceCharacteristics();
								break;
							}
						}
					},
					fail: res => {
						console.log('device services:', res.services)
					}
				});
			},
			/**
			 * 获取指定服务的特征值
			 */
			getBLEDeviceCharacteristics() {
				let deviceId = this.deviceId;
				let serviceId = this.serviceId;
				let characteristicsList = [];
				let self = this;
				uni.getBLEDeviceCharacteristics({
					// 这里的 deviceId 需要已经通过 createBLEConnection 与对应设备建立链接
					deviceId,
					// 这里的 serviceId 需要在 getBLEDeviceServices 接口中获取
					serviceId,
					success: res => {
						console.log('uni.getBLEDeviceCharacteristics - res : ', JSON.stringify(res));
						console.log("获取的" + serviceId + "服务的特征值：" + JSON.stringify(res.characteristics));
						characteristicsList = res.characteristics;
						this.characteristicsLists = res.characteristics;
						for (let i = 0; i < characteristicsList.length; i++) {
							let characteristic = characteristicsList[i];
							//判断服务特征值中的特征值FFF3 和 FFF4
							if (characteristic.uuid.indexOf("FFF3") != -1) {
								self.characteristicId.writeId = characteristic.uuid;
								console.log("设备的特征值写入ID： " + self.characteristicId.writeId);
							}
							if (characteristic.uuid.indexOf("FFF4") != -1) {
								self.characteristicId.notifyId = characteristic.uuid;
								console.log("设备的特征值notifyID： " + self.characteristicId.notifyId);
							}
							if (self.characteristicId.writeId != "" && self.characteristicId.notifyId != "") {
								break;
							}
						}
						this.notifyBLECharacteristicValue();
					},
					fail: res => {
						console.log('device getBLEDeviceCharacteristics failed:', JSON.stringify(res))
					}
				})
			},	
			/**
			 * 开启订阅特征值
			 */
			notifyBLECharacteristicValue() {
				let deviceId = this.deviceId;
				let serviceId = this.serviceId;
				// let characteristicId = this.characteristicId[0].notifyId;
				 let characteristicId = this.characteristicsLists[0].uuid;
				 console.log('notifyBLECharacteristicValue - characteristicId : ', characteristicId);
				// console.log('notifyBLECharacteristicValue - this.characteristicId : ', JSON.stringify(this.characteristicId));
				let notify = true;
				let self = this;
				uni.notifyBLECharacteristicValueChange({
					state: true, // 启用 notify 功能
					// 这里的 deviceId 需要已经通过 createBLEConnection 与对应设备建立链接 
					deviceId,
					// 这里的 serviceId 需要在 getBLEDeviceServices 接口中获取
					serviceId,
					// 这里的 characteristicId 需要在 getBLEDeviceCharacteristics 接口中获取
					characteristicId,
					success(res) {
						console.log('uni.notifyBLECharacteristicValueChange - res : ', JSON.stringify(res));
						// uni.hideLoading();
						uni.showToast({
							title: '连接成功',
							icon: 'success'
						});
						// ArrayBuffer转16进度字符串示例
						function ab2hex(buffer) {
							const hexArr = Array.prototype.map.call(
								new Uint8Array(buffer),
								function(bit) {
									return ('00' + bit.toString(16)).slice(-2)
								}
							)
							return hexArr.join('')
						}
						uni.onBLECharacteristicValueChange(function(res) {
							console.log('uni.onBLECharacteristicValueChange');
							console.log(`characteristic ${res.characteristicId} has changed, now is ${JSON.stringify(res.value)}`)
							var value = ab2hex(res.value);
							let temprrs = value.substr(2);
							self.tempValue = parseInt(temprrs, 16);
							// self.tempValue = 'bb';
							console.log('uni.onBLECharacteristicValueChange - typeof : ', typeof(temprrs));
							// num.toString(16)
							console.log('uni.onBLECharacteristicValueChange - value, tempValue : ', value, self.tempValue);
							// if (value.length === 14){
							// 	let result = parseInt(value.slice(10, 12), 16);
							// 	console.log("动态压力值：" + result);
							// 	self.measureResult[0].pressure = result;
							// }
							// if (value.length === 34){
							// 	let sys = parseInt(value.slice(10, 14), 16);
							// 	let dia = parseInt(value.slice(14, 18), 16);
							// 	let pul = parseInt(value.slice(22, 26), 16);
							// 	console.log("高压值 SYS：" + sys);
							// 	console.log("低压值 DIA：" + dia);
							// 	console.log("脉搏值 PUL：" + pul);
							// 	self.measureResult[0].SYS = sys;
							// 	self.measureResult[0].DIA = dia;
							// 	self.measureResult[0].PUL = pul;
							// 	//停止上传数据
							// 	self.writeBLECharacteristicValue(self.wirteControlCode[0].stopUploadData);
							// }
							// console.log("----------------------------------------------");
						});
					},
					fail(res) {
						console.log('notifyBLECharacteristicValueChange failed:' + res.errMsg);
						// var value = ab2hex(res.value);
						// console.log(value);
					}
				});
			},						
			/**
			 * 停止搜索蓝牙设备
			 */
			stopBluetoothDevicesDiscovery() {
				uni.stopBluetoothDevicesDiscovery({
					success: e => {
						console.log('停止搜索蓝牙设备:' + e.errMsg);
						uni.hideLoading();
						this.isSearch = false;
					},
					fail: e => {
						console.log('停止搜索蓝牙设备失败，错误码：' + e.errCode);
						uni.hideLoading();
						this.isSearch = false;
					}
				});
			},
			/**
			 * 断开蓝牙连接
			 */
			closeBluetooth(device) {
				console.log('断开连接', device);
				uni.closeBLEConnection({
				  deviceId: device.deviceId,
				  success(res) {
				    console.log('断开连接成功', res)
						uni.showToast({
							title: '连接断开',
							icon: 'none'
						})
						device.connectStatus = 0;
				  }
				})
			},
			/**
			 * 断开蓝牙模块
			 */
			closeBluetoothAdapter() {
				uni.closeBluetoothAdapter({
					success: res => {
						console.log('断开蓝牙模块成功');
						this.devicesList = [];
						this.tempValue = 0;
						this.isSearch = '';
						this.isConncet = false;
						uni.showToast({
							icon: "none",
							title: "蓝牙已经断开！",
							mask: false,
							duration: 3000
						});
					}
				});
			}
		},
	}
</script>

<style scoped>
	button.noAllow{
		color: #999;
		background-color: #ccc;
	}
	.device-list{
		position: relative;
		padding: 10rpx;
		border-bottom: 1rpx solid #999999;
	}
	.connect-btn{
		position: absolute;
		top: 30rpx;
		right: 30rpx;
	}
	.ble-act-btn{
		width: 100%;
		padding: 0 20rpx;
		box-sizing: border-box;
		overflow: hidden;
	}
	.ble-act-btn button{
		float: left;
		width: 49%;
	}
	.ble-act-btn button:last-child{
		margin-left: 2%;
	}
</style>
