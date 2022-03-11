<template>
	<view class="content">
		<view v-if="step==-1">
			<view class="cu-form-group margin-top">
				<view class="title">0.系统正在初始化</view>
			</view>
		</view>
		<view v-else-if="step<2">
			<view class="cu-form-group margin-top">
				<view class="title">{{step==0?'1.请您从下边列表中选择一项进行投票,每个密码可以投票一次':'2.现在进入开票阶段,请输入原来的选项与密码来开票'}}</view>
			</view>
			<view class="cu-form-group margin-top">
				<view class="title">选项<text class="text-red">*</text></view>
				<radio-group class="padding-sm flex flex-wrap block" @change="confirmChange">
					<!-- #ifdef MP-ALIPAY -->
					<view v-for="item in values" class="padding-xs">
						<radio :class="vote==item.v?'checked':''" :checked="vote==item.v?true:false" :value="item.v"></radio>{{item.n}}
					</view>
					<!-- #endif -->
					<!-- #ifndef MP-ALIPAY -->
					<view v-for="item in values" class="padding-xs">
						<radio class='radio red margin-left-sm' :checked="vote==item.v?true:false" :value="item.v"></radio>{{item.n}}
					</view>
					<!-- #endif -->
				</radio-group>
			</view>
			<view class="cu-form-group">
				<view class="title">账户<text class="text-red">*</text></view>
				<picker @change="accountChange" class="content" :value="index" :range="accounts">
					<view class="picker">
						{{index==-1?'请先创建账户':accounts[index]}}
					</view>
				</picker>
			</view>
			<view class="cu-form-group margin-top">
				<view class="title">密码<text class="text-red">*</text></view>
				<input placeholder="请输入随即字符串作为密码" password v-model="password" maxlength="128" name="password"></input>
				<text class='cuIcon-roundclose' v-if="password.length" @click="resetpassword"></text>
			</view>
		</view>
		<view v-else>
			<view class="cu-form-group margin-top">
				<view class="title">3.投票已结束,将为您展示投票信息</view>
			</view>
			<view class="cu-form-group">
				<view class="title">{{winner==''?'还未产生':'获胜方'}}</view>
				<view class="content">{{winner}}</view>
			</view>
			<view class="cu-form-group margin-top">
				<view class="title">以下是全部投票信息{{loading?'':('[共'+listData.length+'投票]')}}</view>
			</view>
			<view v-if="listData.length==0" class="basis-xl margin-sm padding-sm empty-list">
				<text>{{loading?'正在获取投票结果':('投票结果为空')}}</text>
			</view>
			<uni-list v-else :margintop="true" :border="true">
				<uni-list-item v-for="item in listData" :key="item" :title="item" />
			</uni-list>
		</view>
		<view v-show="step==0" class="padding-lg">
			<button class="cu-btn block line-green lg" @click="submit"><text class="cuIcon-wenzi"></text>投票</button>
		</view>
		<view v-show="step<2" class="padding-lg">
			<button class="cu-btn block line-green lg" @click="reveal"><text class="cuIcon-close"></text>开票</button>
		</view>
	</view>
</template>

<script>
	var Web3 = require('web3');
	import CommitReveal from '../../contract/CommitReveal.json'
	export default {
		data() {
			return {
				values: [{
					n: 'YES',
					v: '1'
				}, {
					n: 'NO',
					v: '2'
				}],
				accounts: [],
				step: -1,
				password: '',
				contract: null,
				winner: '',
				loading: false,
				index: -1,
				web3: null,
				rpcserver: 'http://localhost:8545',
				contractaddress: '0x3095dFc4BF0b44732899757f40be03BDa96115f4',
				listData: [],
				vote: ''
			}
		},
		onLoad() {
			this.web3 = new Web3(new Web3.providers.HttpProvider(this.rpcserver));
			let that = this
			this.web3.eth.getAccounts(function(err, result) {
				console.log(err, result)
				if (result != null && result != undefined && result.length > 0) {
					that.accounts = result
					that.index = 0
					that.getStatus()
				}
			})
			this.contract = new this.web3.eth.Contract(CommitReveal.abi, this.contractaddress);
		},
		methods: {
			accountChange(e) {
				this.index = e.detail.value
			},
			toast(title) {
				uni.showToast({
					title: title,
					position: 'bottom',
					duration: 2000,
					icon: 'none',
				})
			},
			submit() {
				if (this.password == '') {
					this.toast('请输入密码')
					return
				}
				let v = this.web3.utils.sha3(this.vote + '~' + this.password)
				console.log(v)
				let that = this
				console.log('account:', that.accounts[that.index])
				this.contract.methods.commitVote(v).send({
					from: that.accounts[that.index]
				}, function(err, result) {
					if (err) {
						console.log(that.accounts[that.index], err)
						that.toast('出错了,提示:' + err)
					} else {
						that.toast('已提交到交易池中.')
						that.password = ''
						that.vote = ''
					}
				})
			},
			reveal() {
				if (this.password == '') {
					this.toast('请输入密码')
					return
				}
				let key = this.vote + '~' + this.password
				let v = this.web3.utils.sha3(key)
				console.log(v)
				let that = this
				console.log('account:', that.accounts[that.index])
				this.contract.methods.revealVote(key, v).send({
					from: that.accounts[that.index]
				}, function(err, result) {
					if (err) {
						console.log(that.accounts[that.index], err)
						that.toast('出错了,提示:' + err)
					} else {
						that.toast('已提交到交易池中.')
						that.password = ''
						that.vote = ''
					}
				})
			},
			resetpassword() {
				this.password = ''
			},
			confirmChange(e) {
				this.vote = e.detail.value
			},
			getFinalResult() {
				let that = this
				this.contract.methods.getVoteCommitsArray().call({
					from: that.accounts[that.index]
				}, function(err, result) {
					this.loading = false
					if (err) {
						console.log(that.accounts[that.index], err)
						that.toast('出错了,提示:' + err)
					} else {
						that.listData = result
					}
				})
			},
			getStatus() {
				let that = this
				this.contract.methods.commitPhaseEndTime().call(function(err, result) {
					console.log('commitPhaseEndTime', err, result)
					if (err != undefined || result == undefined || result == null) {
						that.toast('获取合约截止时间出错,提示:' + err)
						return
					}
					let doing = new Date().getTime() < result * 1000
					if (doing) {
						that.step = 0
					}
					console.log('that.canvote ', that.canvote)
					if (!doing) {
						that.contract.methods.getWinner().call({
							from: that.accounts[that.index]
						}, function(err, result) {
							console.log('getWinner', err, result)
							if (err) {
								that.step = 1
								console.log(that.accounts[that.index], err)
							} else {
								that.step = 2
								that.winner = result
								that.getFinalResult()
							}
						})
					}
				})
			}
		}
	}
</script>

<style>

</style>
