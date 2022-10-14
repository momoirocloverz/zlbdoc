<template>
	<view>
		<view class="back">
			<view class="u-flex u-col-bottom">
				<view :class="[uiStyle+'-row-name']" v-if="userInfo.username">
					{{$utils.desensitization(userInfo.username,0)}}
				</view>
				<view :class="[uiStyle+'-row-phone']" v-if="userInfo.mobile">
					{{$utils.desensitization(userInfo.mobile,3,-4)}}
				</view>
			</view>
			<view class="u-m-t-25">
				<u-image height="226" border-radius="30" src="@/assets/home/banner.png" />
			</view>
			<view :class="[uiStyle+'-row-title']">广场舞申请备案</view>
			<view class="u-flex u-m-t-20">
				<view class="u-flex-1 u-m-r-30 u-rela" @click="goto('../apply/application')">
					<view class="u-abso" :class="[uiStyle+'-row-apply']">我要申请</view>
					<u-image border-radius="20" height="200" src="@/assets/home/pic-3.png" />
				</view>
				<view class="u-flex-1 u-rela" @click="goto('../apply/index')">
					<view class="u-abso" :class="[uiStyle+'-row-apply']">我的申请</view>
					<u-image border-radius="20" height="200" src="@/assets/home/pic-4.png" />
				</view>
			</view>
			<view :class="[uiStyle+'-row-title']">噪音投诉</view>
			<view class="u-flex u-m-t-20">
				<view class="u-flex-1 u-flex u-flex-col" :class="[uiStyle+'-row-complaint',index?'':'u-m-r-30']"
					v-for="(item,index) in homeList" @click="goto(item.path)">
					<view class="complaint-back">
						<u-icon :name="item.img" color="#D29C50" custom-prefix="noise-icon" size="55" />
					</view>
					<view class="complaint-title">{{item.title}}</view>
				</view>
			</view>
			<view :class="[uiStyle+'-row-title']">噪音监控</view>
			<view class="u-flex u-flex-1 u-rela row-monitor" @click="goto('../monitor/index')">
				<view :class="[uiStyle+'-monitor-title']">监控设备</view>
				<u-image class="monitor-img" width="230" height="170" src="@/assets/home/pic-5.png" />
			</view>
		</view>
		<view class="u-font-24 u-m-t-50 u-m-b-30 u-flex u-row-center" style="flex-direction:column;color: #7d7d7d;">
			<view>本服务由浙江政务服务网、舟山市普陀区综合行政执法局提供</view>
			<view class="u-flex u-m-t-5">
				<view>服务咨询热线：</view>
				<view @click="callPhone" style="color: #0898f8;">0580-3053333</view>
			</view>
		</view>
	</view>
</template>

<script>
	import {
		mgop
	} from '@aligov/jssdk-mgop';
	import md5Libs from "uview-ui/libs/function/md5";
	let that = null;
	export default {
		data() {
			return {
				ticket: '',
				uiStyle: 'normal',
				userInfo: {
					idnum: '',
					mobile: '',
					userid: '',
					username: ''
				},
				homeList: [{
					title: '我要投诉',
					img: 'phone',
					path: '../complaint/complaint'
				}, {
					title: '我的投诉',
					img: 'people',
					path: '../complaint/index'
				}],
				userType: null,
				longitud: null,
				latitude: null
			}
		},
		onLoad(options) {
			that = this;
			that.ticket = options.ticket;
			that.init();
		},
		onShow() {
			ZWJSBridge.setTitle({
				title: '广场舞一件事'
			});
		},
		methods: {
			init: () => {
				ZWJSBridge.setTitle({
					title: '广场舞一件事'
				});
				ZWJSBridge.getUiStyle().then(res => {
					switch (res.uiStyle) {
						case 'elder':
							that.uiStyle = res.uiStyle;
							break;
						default:
							that.uiStyle = 'normal';
					}
					uni.setStorageSync('uiStyle', that.uiStyle);
				});
				that.getTonkenAndUserInfo(that.getTonkenAndUserInfoParams()).then(data => {
					that.userInfo.username = data.username;
					that.userInfo.userid = data.userid;
					return that.getTonkenAndUserInfo(that.getTonkenAndUserInfoParams('getUserInfo', data
						.token), 'getUserInfo');
				}).then(data => {
					that.userInfo.idnum = data.idnum;
					that.userInfo.mobile = data.mobile;
					return ZWJSBridge.getUserType({});
				}).then(res => {
					that.userType = res.userType;
					return ZWJSBridge.getLocation();
				}).then(res => {
					that.longitude = res.longitude;
					that.latitude = res.latitude;
					uni.setStorageSync('userInfo', that.userInfo);
					that.aplus();
					return that.$request.addUser(that.userInfo);
				}).then(data => {
					uni.setStorageSync('token', data.msg);
				}).catch(err => {
					console.log(err);
				});
			},
			aplus: () => {
				aplus_queue.push({
					action: 'aplus.sendPV',
					arguments: [{
						is_auto: false
					}, {
						miniAppId: '2001907982',
						miniAppName: '广场舞一件事',
						long: that.longitude,
						lati: that.latitude,
						userType: that.userType,
					}]
				});
				aplus_queue.push({
					action: 'aplus.setMetaInfo',
					arguments: ['_user_nick', that.userInfo.username]
				});
				aplus_queue.push({
					action: 'aplus.setMetaInfo',
					arguments: ['_user_id', that.userInfo.userid]
				});
				aplus_queue.push({
					action: 'aplus.setMetaInfo',
					arguments: ['_hold', 'START']
				});
			},
			goto: (value) => {
				uni.navigateTo({
					url: value
				})
			},
			getTonkenAndUserInfo: (data, method = 'ticketValidation') => {
				return new Promise((resolve, reject) => {
					mgop({
						api: `mgop.zjzwfw.noise.${method}`,
						host: 'https://mapi.zjzwfw.gov.cn/',
						data: data,
						dataType: 'JSON',
						type: 'POST',
						appKey: '5h1w38we+2001907982+dgvrks',
						onSuccess: res => {
							if (res.data.result && res.data.result == 0) {
								resolve(res.data);
							}
						},
						onFail: err => {
							reject(err);
						}
					});
				})
			},
			getTonkenAndUserInfoParams: (method = 'ticketValidation', token = '') => {
				let mTime = that.$u.timeFormat(new Date(), 'yyyymmddhhMMss');
				let data = {
					method: method,
					servicecode: 'gcwyjs',
					time: mTime,
					sign: md5Libs.md5(`gcwyjsgcwyjspwd${mTime}`),
					datatype: 'json'
				}
				if (token == '') {
					data.st = that.ticket;
				} else {
					data.token = token;
				}
				return data;
			},
			callPhone: () => {
				ZWJSBridge.phoneCall({
					corpId: '0580-3053333'
				}).then(res => {
					console.log(res)
				}).catch(err => {
					console.log(err)
				})
			}
		}
	}
</script>

<style lang="scss">
	@import "common/css/uiStyle.scss";

	page {
		background-color: #EAEAEA;
	}

	.back {
		background-color: #FFF;
		padding: 20rpx 20rpx 50rpx;
	}

	.row-monitor {
		overflow: hidden;
		background-color: #E2FFF2;
		height: 200rpx;
		border-radius: 20rpx;
		width: 48.5%;

		.monitor-img {
			position: absolute;
			right: -40rpx;
			bottom: 0;
		}
	}

	@each $uiStyle in (normal, elder) {
		.#{$uiStyle}-monitor-title {
			@include zlb-uiStyle(36, #59A6B8, if(getUiStyle($uiStyle), 600, bold)) {
				margin-left: 30rpx;
			}
		}

		@each $name in (name, phone, title, apply, complaint) {
			.#{$uiStyle}-row-#{$name} {
				@if $name == name {
					@include zlb-uiStyle(if(getUiStyle($uiStyle), 43, 54), #2A2E43, if(getUiStyle($uiStyle), 400, bold));
				} @else if $name == phone {
					@include zlb-uiStyle(if(getUiStyle($uiStyle), 31, 42), #78849E, 400) {
						margin-left: 12rpx;
					}
				} @else if $name == title {
					@include zlb-uiStyle(if(getUiStyle($uiStyle), 42, 56), #2A2E43, if(getUiStyle($uiStyle), 400, bold)) {
						margin: 40rpx 0 20rpx;
					}
				} @else if $name == apply {
					@include zlb-uiStyle(if(getUiStyle($uiStyle), 36, 44), #0B508D, if(getUiStyle($uiStyle), 600, bold)) {
						z-index: 99;
						top: 15rpx;
						left: 30%;
					}
				} @else {
					background-color: #F9F5F0;
					height: 200rpx;
					border-radius: 20rpx;

					.complaint-back {
						$margin: 15;
						background-color: #FFFFFF;
						border-radius: 50%;
						padding: 20rpx;
						width: 100rpx;
						text-align: center;

						@if getUiStyle($uiStyle) {
							$margin: $margin+5;
						}

						margin: #{$margin}rpx 0;
					}

					.complaint-title {
						@include zlb-uiStyle(if(getUiStyle($uiStyle), 36, 44), #8A775C, 600);
					}
				}
			}
		}
	}
</style>
