<template>
	<div>
		<div class="error window half-width info" id="error-window" v-show='error'>
          {{error}}
        </div>

	    <total-balances/>

		<div class="window white" v-for='(currency, i) in Object.keys(filteredCurrencies)' :key="i">
			  <p class='simple-error' v-show="currency == 'susd'"> Old susd pool. Please <router-link to="/susd/withdraw">withdraw and move</router-link> funds to <router-link to="/susdv2">susdv2</router-link> pool </p>
		      <p class='text-center'>
		      	<router-link :to="currency" v-show="currency != 'susd' && !['tbtc', 'ren', 'sbtc'].includes(currency)">{{currency == 'iearn' ? 'y' : currency}}.curve.fi</router-link>
	      		<span v-show="['tbtc', 'ren', 'sbtc'].includes(currency)">{{currency}}</span>
		      	<a href='https://iearn.finance/pool' v-show="currency == 'susd'">susd</a>
		      </p>
		      <stats :pool= 'currency'/>
		      <balances-info
			      :bal_info = 'bal_infos[currency]'
			      :total = 'totals[i]'
			      :l_info = 'l_infos[currency]'
			      :totalShare = 'totalShares[i]'
			      :fee = 'fees[i]'
			      :admin_fee = 'admin_fees[i]'
			      :pool = 'currency'
			      :currencies = 'allCurrencies[currency]'
			      :tokenSupply = 'totalTokenSupplies[i]'
			      :tokenBalance = 'totalTokenBalances[i]'
			      :usdShare = 'usdShares[i]'
			      :staked_info = "['susdv2', 'sbtc', 'y', 'iearn'].includes(currency) && staked_infos[currency]"
			      :totalStake = 'totalStakes[currency]'
			      :usdStake = 'usdStakes[currency]'
			      :virtual_price = 'virtual_prices[i]'
			      :A = 'As[i]'
			      :future_A = 'future_As[i]'
			      :admin_actions_deadline = 'admin_actions_deadlines[i]'
			      :combinedstats = 'true'
			      />
	  	</div>
	</div>
</template>

<script>
	import Vue from 'vue'
	import Stats from '../Stats.vue'
	import BalancesInfo from '../BalancesInfo.vue'
	import Web3 from 'web3'
	import TotalBalances from './TotalBalances.vue'

    import { getters, contract as currentContract, allCurrencies } from '../../contract'
    import contracts, { infura_url, ERC20_abi, cERC20_abi, yERC20_abi } from '../../allabis'

    import * as helpers from '../../utils/helpers'

	export default {
		metaInfo: {
	      title: 'Swerve Tron - Stats',
	      meta: [
	      	{'property': 'og:url', 'content': 'https://curve.fi/combinedstats'},
	      	{'name': 'twitter:url', 'content': 'https://curve.fi/combinedstats'},
	      ]
	    },
		components: {
			Stats,
			BalancesInfo,
			TotalBalances,
		},
		data: () => ({
			contracts: [],
			web3contracts: {},
			all_coins: {},
			all_underlying_coins: {},
			all_c_rates: {},
			all_fees: {},
			totals: [],
			bal_infos: {},
			l_infos: {},
			totalShares: [],
			totalStakes: {},
			usdShares: [],
			usdStakes: {},

			staked_infos: {},
			totalStake: -1,
			usdStake: -1,

			virtual_prices: [],
			As: [],
			future_As: [],
			admin_actions_deadlines: [],

			fees: [],
			admin_fees: [],
			totalTokenBalances: [],
			totalTokenSupplies: [],
		}),
		created() {
            this.$watch(()=>currentContract.initializedContracts, val => {
                if(val) this.mounted();
            })
            this.$watch(()=>currentContract.currentContract, val => {
            	if(currentContract.initializedContracts) this.mounted();
            })
        },
        computed: {
          allCurrencies() {
          	return Object.assign(allCurrencies, {
          		susd: {
          			susd: 'sUSD',
          			ycurve: 'yCurve'
          		}
          	})
          },
          allContracts() {
          	let pools = {...contracts};
          	delete pools.y
          	delete pools.tbtc
          	return pools;
          },
          error() {
          	return currentContract.error
          },
          filteredCurrencies() {
          	return Object.fromEntries(Object.entries(this.allCurrencies).filter(
			      ([key, val])=>!['tbtc'].includes(key)
			   ))
          }
        },
        mounted() {
            if(currentContract.initializedContracts) this.mounted();
        },
		methods: {
			async mounted() {
				this.fees = Array.from(4).fill(0)
				this.admin_fees = Array.from(4).fill(0)
				this.admin_fees = Array.from(4).fill(0)
				this.admin_fees = Array.from(4).fill(0)
				await this.init_contracts();
				await this.update_fee_info();
			},
			poolLink(pool) {
				if(pool == 'iearn') return 'y'
				if(pool == 'susd') return 'https://iearn.finance/pool'
				return pool
			},
			async init_contracts() {
			    for(let [key, contract] of Object.entries(this.allContracts)) {
			    	this.web3contracts[key] = {};
				    this.web3contracts[key].swap = new currentContract.web3.eth.Contract(contract.swap_abi, contract.swap_address);
				    this.web3contracts[key].swap_token = new currentContract.web3.eth.Contract(ERC20_abi, contract.token_address);

			        this.all_coins[key] = {}
			        this.all_coins[key].coins = [];
			        this.all_underlying_coins[key] = {}
			        this.all_underlying_coins[key].underlying_coins = [];
				    for (let i = 0; i < contract.N_COINS; i++) {
				        var addr = contracts[key].coins[i];

				        let cabi = ['iearn', 'y', 'busd', 'susd', 'pax'].includes(key) ? yERC20_abi : cERC20_abi;
				        if(key == 'susd' && i == 1) {
				        	cabi = contracts.iearn.swap_abi;
				        	addr = contracts.iearn.swap_address
				        }
				        this.all_coins[key].coins[i] = new currentContract.web3.eth.Contract(cabi, addr);
				        var underlying_addr = contracts[key].underlying_coins[i];
				        this.all_underlying_coins[key].underlying_coins[i] = new currentContract.web3.eth.Contract(ERC20_abi, underlying_addr);
				    }
			    }
			},

			async update_rates() {
		        let calls = []
			    for(let [key, contract] of Object.entries(this.allContracts)) {
			    	this.all_fees[key] = [];
			     	this.all_c_rates[key] = {}
			        this.all_c_rates[key].c_rates = [];
				    for (let i = 0; i < contract.N_COINS; i++) {
				        /*
				        rate: uint256 = cERC20(self.coins[i]).exchangeRateStored()
				        supply_rate: uint256 = cERC20(self.coins[i]).supplyRatePerBlock()
				        old_block: uint256 = cERC20(self.coins[i]).accrualBlockNumber()
				        rate += rate * supply_rate * (block.number - old_block) / 10 ** 18
				        */
			         	if (contract.tethered && contract.tethered[i]
			         		&& contract.use_lending && !contract.use_lending[i]
			         		|| key == 'susdv2' || (key == 'pax' && i == 3) || key == 'tbtc' || key == 'ren' || key == 'sbtc') {
			            	this.all_c_rates[key].c_rates[i] = 1 / contract.coin_precisions[i]
			         	}
			         	else {
			         		if(key == 'iearn' || key == 'y' || key == 'busd' || (key == 'susd' && i == 0) || (key == 'pax' && i < 3)) {
			            		calls.push([
			            			this.all_coins[key].coins[i]._address,
			            			this.all_coins[key].coins[i].methods.getPricePerFullShare().encodeABI()
			            		])
			         		}
			         		else if(key == 'susd' && i == 1) {
			         			calls.push([
			         				this.all_coins[key].coins[i]._address,
			         				this.all_coins[key].coins[i].methods.get_virtual_price().encodeABI()
			         			])
			         		}
			         		else {
						        if(key != 'usdt') {
							        calls.push(
							        	[
							        		this.all_coins[key].coins[i]._address,
							        		this.all_coins[key].coins[i].methods.exchangeRateStored().encodeABI()
							        	],
							        	[
							        		this.all_coins[key].coins[i]._address,
							        		this.all_coins[key].coins[i].methods.supplyRatePerBlock().encodeABI()
							        	],
							        	[
							        		this.all_coins[key].coins[i]._address,
							        		this.all_coins[key].coins[i].methods.accrualBlockNumber().encodeABI()
							        	],
							        )
						    	}
			         		}
					    }
					    calls.push([
					    	this.web3contracts[key].swap._address,
					    	this.web3contracts[key].swap.methods.balances(i).encodeABI(),
					    ])
				    }
				    calls.push(
				    	[
				    		this.web3contracts[key].swap._address,
				    		this.web3contracts[key].swap.methods.get_virtual_price().encodeABI(),
				    	],
				    	[
				    		this.web3contracts[key].swap._address,
				    		this.web3contracts[key].swap.methods.A().encodeABI(),
				    	],
				    	[
				    		this.web3contracts[key].swap._address,
				    		this.web3contracts[key].swap.methods.future_A().encodeABI(),
				    	],
				    	[
				    		this.web3contracts[key].swap._address,
				    		this.web3contracts[key].swap.methods.admin_actions_deadline().encodeABI(),
				    	],
				    	[
				    		this.web3contracts[key].swap._address,
				    		this.web3contracts[key].swap.methods.fee().encodeABI(),
				    	],
				    	[
				    		this.web3contracts[key].swap._address,
				    		this.web3contracts[key].swap.methods.admin_fee().encodeABI(),
				    	],
				    	[
				    		this.web3contracts[key].swap_token._address,
				    		this.web3contracts[key].swap_token.methods.balanceOf(currentContract.default_account || '0x0000000000000000000000000000000000000000').encodeABI(),
				    	],
				    	[
				    		this.web3contracts[key].swap_token._address,
				    		this.web3contracts[key].swap_token.methods.totalSupply().encodeABI(),
				    	],
				    )
				}
				return calls;
			},

			async update_fee_info(version = 'new') {
				await Promise.resolve();
			    // let calls = await this.update_rates();
				// calls.push([contracts.susdv2.sCurveRewards_address, '0x70a08231000000000000000000000000' + (currentContract.default_account || '0x0000000000000000000000000000000000000000').slice(2)])
				// calls.push([contracts.sbtc.sCurveRewards_address, '0x70a08231000000000000000000000000' + (currentContract.default_account || '0x0000000000000000000000000000000000000000').slice(2)])
				// calls.push([contracts.iearn.sCurveRewards_address, '0x70a08231000000000000000000000000' + (currentContract.default_account || '0x0000000000000000000000000000000000000000').slice(2)])
			    // let aggcalls = await currentContract.multicall.methods.aggregate(calls).call();
			    // let block = aggcalls[0]
			    // let decoded = aggcalls[1].map(hex => currentContract.web3.eth.abi.decodeParameter('uint256', hex))
			    // let curveStakedBalance = decoded[decoded.length-3]
			    // let sbtcCurveStakedBalance = decoded[decoded.length-2]
			    // let yCurveStakedBalance = decoded[decoded.length-1]
			    // console.log(yCurveStakedBalance, "Y CURVE STAKED BALANCE")
			    // decoded = decoded.slice(0, decoded.length-1)
			    // let i = 0;
			    // this.bal_infos['usdt'] = []
			    // this.l_infos['usdt'] = []
				// for(let [key, contract] of Object.entries(this.allContracts)) {
				// 	this.bal_infos[key] = []
				// 	this.l_infos[key] = []
				// 	var total = 0;
				// 	let ind = i*16;
				// 	if(i > 1) ind-=5;
				//     if(key == 'iearn' || key == 'busd' || key == 'susd' || key == 'susdnew') {
				//     	let slice = decoded.slice(ind, ind+contracts[key].N_COINS*2)
				//     	helpers.chunkArr(slice, 2).map((v, i) => {
				//     		//v is [rate, balance] or just [balance] for PAX in pax pool
				//     		if(v[1] === undefined) {
				//     			let balance = +v[0]
				//     			let calcBalance = this.all_c_rates[key].c_rates[i] * balance
				//     			this.bal_infos[key].push(calcBalance)
				//     			total += calcBalance
				//     		}
				//     		else {
				// 	    		let rate = +v[0] / 1e18 / contracts[key].coin_precisions[i]
				// 	    		if(key == 'susd' && i == 1) rate = +v[0] / 1e36
				// 	    		this.all_c_rates[key].c_rates[i] = rate
				// 	    		let balance = +v[1]
				// 	    		let calcBalance = rate*balance
				// 	    		this.bal_infos[key].push(calcBalance)
				// 	    		total += calcBalance
				//     		}
				//     	})
				//     	if(key == 'susd') ind -= 4
				//     }
				//     this.totals.push(total)
				//     this.virtual_prices.push(+decoded[ind+8] / 1e18)
				//     this.As.push(+decoded[ind+9])
				//     this.future_As.push(+decoded[ind+10])
				//     this.admin_actions_deadlines.push(+decoded[ind+11])


				//     this.fees.push(+decoded[ind+12] / 1e8)
		    	// 	this.admin_fees.push(+decoded[ind+13])
		    	// 	this.totalTokenBalances.push(+decoded[ind+14])
		    	// 	this.totalTokenSupplies.push(+decoded[ind+15])
		    	// 	var totalShare = 0
				//     for (let i=0; i < contracts[key].N_COINS; i++) {
		        //         var val = this.bal_infos[key][i] * (+decoded[ind+14]) / (+decoded[ind+15]);
		        //         this.l_infos[key].push(val)
		        //         totalShare += val;
		        //     }
		        //     this.usdShares.push(decoded[ind+14] * decoded[ind+8] / 1e36)
	            // 	this.totalShares.push(totalShare)
	            // 	if(['susdv2', 'sbtc', 'y', 'iearn'].includes(key)) {
	            // 		let stakedBalance = curveStakedBalance
	            // 		if(key == 'sbtc') stakedBalance = sbtcCurveStakedBalance
	            // 		if(['y', 'iearn'].includes(key)) stakedBalance = yCurveStakedBalance
	            // 		this.totalStakes[key] = 0
	            // 		this.staked_infos[key] = []
	            // 		if(stakedBalance > 0) {
	            // 			for(let i=0; i < contracts[key].N_COINS; i++) {
	            // 				var val = this.bal_infos[key][i] * stakedBalance / (+decoded[ind+15]);
	            // 				Vue.set(this.staked_infos[key], i, val)
	            // 				this.totalStakes[key] += val
	            // 			}
	            // 		}
	            // 		this.usdStakes[key] = stakedBalance * decoded[ind+8] / 1e36
	            // 	}

	            // 	i++;
				// }
			},


		}
	}
</script>