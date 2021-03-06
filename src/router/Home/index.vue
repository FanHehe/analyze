<template>
	<div class = "home-container">
		<Navigator />
		<div class = "main" :style = "{ top: -state.index * height + 'px' }">
			<section class = "section" id = "sec-1" ref = "canvas">
				<h1>股票分析系统</h1>
			</section>
			<section class = "section" id = "sec-0">
				<div class = "section-search">
					<Search
						:dropdown = "nameToStockInfoDropdown"
						:handleInput = "nameToStockInfoSubmit"
						:handleSubmit = "nameToStockInfoSubmit"
						:handleClickItem = "nameToStockInfoClickItem" />
				</div>
			</section>
			<section class = "section" id = "sec-2">
				<div class = "sec-2-filter">
					<div class = "drop-down">
						<span>
							{{ state.curMarket ? `${ state.curMarket.short.toUpperCase() }（${ state.curMarket.name}）` : '您要选哪个股市'}}
						</span>
						<ul>
							<li
								v-for = "(item, index) in stock.markets"
								@click = "handleClickSelectMarket(item)"
								:key = "index">{{ `${item.short.toUpperCase()}(${item.name})` }}</li>
						</ul>
					</div>
				</div>
				<div class = "sec-2-content">
					<div class= "sec-2-list" v-if = "indexListSearch.length">
						<ul>
							<span>\</span>
							<li>市场</li>
							<li>名字</li>
							<li>代码</li>
							<li>拼音</li>
						</ul>
						<ul v-for = "(row, rowIndex) in indexListSearch.slice((state.indexSearchPage - 1) * numPerPage, state.indexSearchPage * numPerPage)"
							:key = "rowIndex">
							<span>
								{{ (rowIndex + 1) + (indexListSearch.currentPage - 1) * indexListSearch.maxResult + (state.indexSearchPage - 1) *numPerPage }}
							</span>
							<li v-for = "(col, colIndex) in row"
								:key = "colIndex" 
                                @click = "handleClickTableRow(row)"
                                >
								{{ col }}
							</li>
						</ul>
					</div>
					<div class = "sec-2-info">
						<div class = "sec-2-page-wrap">
							<Page
								:pageLimit = "7"
								:maxPage = "Math.floor(indexListSearch.allNum / numPerPage)"
								:curPage = "(indexListSearch.currentPage - 1) * indexListSearch.maxResult / numPerPage + state.indexSearchPage"
								:handleClickPageItem = "clickIndexListSearchPageItem"
								/>
						</div>
					</div>
				</div>
			</section>
			<section class = "section" id = "sec-3">
			</section>
		</div>
		<div class = "main-list">
			<a v-for = "(item, index) in newList(n)"
				:class = "{ active: item === state.index }"
				@click = "clicPagePoint(item)"
				:key = "index"></a>
		</div>
		<div class = "main-slider">
			<ul>
				<a href="javascript:void(0)">股指列表查询</a>
				<a href="javascript:void(0)">股指实时分时线查询</a>
				<a href="javascript:void(0)">股票实时K线数据查询</a>
				<a href="javascript:void(0)">股票实时分时线数据查询</a>
				<a href="javascript:void(0)">沪深及港股历史行情查询</a>
				<a href="javascript:void(0)">名称编码拼音查询股票信息查询</a>
				<a href="javascript:void(0)">股票列表查询</a>
				<a href="javascript:void(0)">沪深股票最新50条逐笔交易查询</a>
			</ul>
		</div>
		<Login />
	</div>
</template>
<script>

	import { mapState, mapGetters } from 'vuex';

	import Page from '../../components/Page';
	import Search from '../../components/Search';
	import MyTable from '../../components/MyTable';

	import { STOCK } from '../../common/constants';

	import Canvas from './Canvas';
	import Point, { distance } from './Point';

	import Login from '../../components/Login';
	import Navigator from '../../components/Navigator';

	export default {
		data () {
			return {
				n: 4, // 总共的页数
				page: 1, // 当前页
				numPerPage: 10, // 每页最多显示10条数据
				chart: null,
				height: window.innerHeight,
				state: {
					index: 0,
					curMarket: null,
					isTurning: false,
					indexSearchPage: 1,
					loadingNameToStockInfo: false,
				},
				stock: {
					markets: [{ short: 'sh', name: '上海' }, { short: 'sz', name: '深圳' }, { short: 'hk', name: '香港'}],
				}
			};
		},
		computed: {
			...mapState([
				'indexListSearch',
				'indexTimeLine',
				'indexList',
				'realTimeK',
				'timeline',
				'history',
				'kLine',
				'realStockInfo',
				'batchRealStockInfo',
				'nameToStockInfo',
				'stockList',
				'recentTrade'
			]),
			...mapGetters([
				'nameToStockInfoDropdown',
			]),
		},
		components: { Page, MyTable, Search, Navigator, Login },
		methods: {
			nameToStockInfoSubmit (content) {
				console.log(this.state.loadingNameToStockInfo, content);
				if (this.checkLoading('loadingNameToStockInfo')) return ;
				const query = {};
				const numReg = /^[0-9]{3,6}$/;
				const pinyinReg = /^[a-zA-Z]{1,8}$/;
				const chineseReg = /[\u4E00-\u9FA5]/g;
				if (numReg.test(content)) query.code = content;
				else if (pinyinReg.test(content)) query.pinyin = content;
				else if (chineseReg.test(content)) query.name = content;
				else return;
				this.getNameToStockInfo(query);
			},
			nameToStockInfoClickItem (item) {
				const { code } = item;
				const path = `/stock/${code}`;
				this.$router.push(path);

			},
			clicPagePoint (item) {
				if (item === this.state.index) return ;
				this.state.index = item;
			},
			handleMouseWheel (event) {
				if (this.state.isTurning) return;
				this.state.isTurning = true;
				setTimeout(() => { this.state.isTurning = false; }, 1500);
				const { wheelDelta } = event;
				if (wheelDelta >= 0) this.turnPage(false); // 向上翻页
				else this.turnPage(true); // 向上翻页
			},
			handleClickSelectMarket (item) {
				const page = 1;
				const market = item.short;
				this.state.curMarket = item;
				this.getIndexListSearch({ page, market });
			},
			clickIndexListSearchPageItem (index, oldIndex) {
				const config = {};
				const { numPerPage } = this;
				const { curMarket } = this.state;
				const { currentPage, maxResult } = this.indexListSearch;

				const nPart = Math.floor(maxResult / numPerPage);
				const rest = (index - 1) % nPart + 1;
				const divide = Math.floor((index - 1) / nPart);

				const success = () => { this.state.indexSearchPage = rest; };
				if (divide !== Math.floor(oldIndex / nPart )) {
					this.getIndexListSearch({ page: divide + 1, market: curMarket, success });
				} else {
					this.state.indexSearchPage = rest;
				}
			},
			turnPage (signal) {
				const { n } = this;
				let { index } = this.state;
				index = signal === false ? index - 1 : index + 1;
				index = Math.min(index, n - 1);
				index = Math.max(index, 0);
				this.state.index = index;
			},
			newList (n) {
				return new Array(n).fill(n).map((item, index) => index);
			},
			// 股指列表查询
			getIndexListSearch (data) {
				const { page, market, success } = data;
				const { maxResult } = 10;
				this.publish(STOCK.STOCK_INDEX_SEARCH.name, { success, query: { page, market, maxResult }});
			},
			// 股指实时分时线
			getIndexTimeLine () {
				const day = 1; // 1 - 5
				const code = '000001';
				this.publish(STOCK.STOCK_INDEX_TIMELINE.name, { query: { day, code }});
			},
			// 股指实时行情_批量
			getIndexList () {
				const stocks = 'sh000001,sz399001,sz399005,sz399006,hkhsi';
				this.publish(STOCK.STOCK_INDEX.name, { query: { stocks }});
			},
			// 股票实时K线数据
			getRealTimeK () {
				const beginDay = '20160316';
				const code = '000001';
				const time = 'week'; // '5' || '30' || '60' || 'day' || 'week' || 'month'
				const type = 'bfq'; // 'bfq' || 'qff'
				this.publish(STOCK.STOCK_REALTIME_K.name, { query: { beginDay, code, time, type }});
			},
			// 股票实时K线数据
			getTimeline () {
				const day = '1';
				const code = '601857';
				this.publish(STOCK.STOCK_TIMELINE.name, { query: { day, code }});
			},
			// 沪深及港股历史行情
			getHistory () {
				const begin = '2017-03-14';
				const code = '600004';
				const end = '2017-03-15';
				this.publish(STOCK.STOCK_SZ_SH_STOCK_HISTORY.name, { query: { begin, code, end } });
			},
			// 股指实时K线数据
			getKLine () {
				const beginDay = '20170315';
				const code = '000001';
				const time = 'day'; // '5' || '30' || '60' || 'day' || 'week' || 'month'
				this.publish(STOCK.STOCK_INDEX_K_LINE.name, { query: { time, code, beginDay }});
			},
			// 股票实时行情
			getRealStockInfo () {
				const code = '000001';
				const needIndex = 0; // 0 || 1;
				const needKPic = 0; // 0 || 1;
				this.publish(STOCK.STOCK_REAL_STOCK_INFO.name, { query: { code, needIndex, need_k_pic: needKPic}});
			},
			// 股票实时行情_批量
			getBatchRealStockInfo () {
				const needIndex = 0; // 0 || 1;
				const stocks = 'sh000001,sz399001,sz399005,sz399006,hkhsi';
				this.publish(STOCK.STOCK_BATCH_REAL_STOCK_INFO.name, { query: { stocks, needIndex }});
			},
			// 名称编码拼音查询股票信息
			getNameToStockInfo ({ code, name, pinyin }) {
				const query = {};
				if (code) query.code = code;
				else if (name) query.name = name;
				else if (pinyin) query.pinyin = pinyin;
				else return;
				this.publish(STOCK.STOCK_NAME_TO_STOCK_INFO.name, { query });
			},
			// 股票列表查询
			getStockList () {
				const market = 'sz';
				const page = '1';
				this.publish(STOCK.STOCK_LIST.name, { query: { market, page }});
			},
			// 沪深股票最新50条逐笔交易
			getRecentTrade () {
				const code = '000002';
				this.publish(STOCK.STOCK_EVERY_TRADE.name, { query: { code }});
			},
			publish (type, payload) {
				const { dispatch } = this.$store;
				dispatch({ ... payload, type });
			},
			draw (ctx, pointes) {
				var _self = window;
				var mX = _self.mX;
				var mY = _self.mY;

				var width = ctx.canvas.width;
				var height = ctx.canvas.height;
				ctx.clearRect(0, 0, width, height);
				for (var i = 0, pLen = pointes.length; i < pLen; i ++) {
					pointes[i].move();
					pointes[i].draw(ctx);
					var len = distance(pointes[i].x, pointes[i].y, _self.mX, _self.mY);
					if ( len < 150) {
						ctx.beginPath();
						ctx.lineWidth = 1 - len / 150;
						ctx.strokeStyle = pointes[i].color;
						ctx.moveTo(mX, mY);
						ctx.lineTo(pointes[i].x, pointes[i].y);
						ctx.closePath();
						ctx.stroke();
					}
				}
				for ( var i = 0, len = pointes.length; i < len; i++) {
					for (var j = i + 1, pLen = pointes.length; j < pLen; j++) {
						var len = distance(pointes[i].x, pointes[i].y, pointes[j].x, pointes[j].y);
						if ( len < 150) {
							ctx.beginPath();
							ctx.lineWidth = 1 - len / 150;
							ctx.strokeStyle = pointes[i].color;
							ctx.moveTo(pointes[j].x, pointes[j].y);
							ctx.lineTo(pointes[i].x, pointes[i].y);
							ctx.closePath();
							ctx.stroke();
						}
					}
				}
				window.requestAnimationFrame(this.draw.bind(window, ctx, pointes));
			},
			initCanvas () {
				const cvs = new Canvas();
				cvs.insertInto(this.$refs.canvas);
				var pointes = [];
				var width = cvs.canvas.width;
				var height = cvs.canvas.height;
				for (var i = 0; i < 0.0001 * width * height; i++) pointes.push(new Point(width, height));
				cvs.canvas.addEventListener('mousemove', function (e) { window.mX = e.clientX; window.mY = e.clientY; });
				this.draw.call(window, cvs.context, pointes);
			},
			checkLoading (key, timeout = 0) {
				if (this.state[key]) return true;
				this.state[key] = true;
				setTimeout(() => { this.state[key] = false; });
			},
            handleClickTableRow ({ code }) {
                this.$router.push(`/stock/${ code }`);
            }
		},
		mounted () {
			const chart = this.$refs.chart;
			this.initCanvas();
			// 测试api
			// this.getIndexTimeLine();
			// this.getIndexList();
			// this.getRealTimeK();
			// this.getTimeline();
			// this.getHistory();
			// this.getKLine();
			// this.getRealStockInfo();
			// this.getBatchRealStockInfo();
			// this.getNameToStockInfo();
			// this.getStockList();
			// this.getRecentTrade();
			window.addEventListener('mousewheel', this.handleMouseWheel.bind(this));
		},
		beforeDestory () {
			window.removeEventListener('mousewheel', this.handleMouseWheel);
		},
	};
</script>
<style lang = "scss">
	.home-container { width: 100%; height: 100%; overflow: hidden; position: relative; }
	.main { width: 100%; height: 100%; position: absolute; transition: 500ms cubic-bezier(0.86, 0, 0.07, 1); }
	.section { width: 100%; height: 100%; display: block; color: #FFFFFF; position: relative; overflow: hidden; padding: 20px; box-sizing: border-box; }
	.main-list {
		width: 20px;
		right: 20px;
		height: 100%;
		display: flex;
		position: fixed;
		align-items: center;
		flex-direction: column;
		justify-content: center;

		a {
			width: 10px;
			height: 10px;
			margin: 10px 0;
			display: block;
			cursor: pointer;
			border-radius: 50%;
			border: 1px solid #ffffff;
		}
		.active { background-color: #ffffff; }
	}
	.main-slider {
		top: 0;
		left: 0;
		width: 0;
		z-index: 1;
		height: 100%;

		font-size: 10px;
		font-weight: 900;
		max-width: 400px;
		visibility: hidden;
		position: absolute;
		background: #ffffff;
		box-sizing: border-box;
		.active { width: 30%; min-width: 200px; visibility: visible; }
		a { display: block; width: 100%; height: 30px; font-size: 2em; padding: 20px ; box-sizing: border-box; }
		a.active { background: #ffffff; color: #000000; }
	}
	#sec-1 {
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 0;
		position: relative;

		canvas { position: absolute; z-index: 0; }

		h1 {
			margin: 0;
			z-index: 1;
			font-size: 100px;
			position: relative;
			margin-top: -200px;
			box-sizing: border-box;
			text-shadow: 1px 1px 3px #ffffff, 2px 2px 7px #ffffff, 3px 3px 10px #ffffff, 5px 5px 15px #000000;
		}
	}
	.section-search {
		margin-top: 100px;
	}
	.sec-2-filter {
		margin: 80px 0 40px;
		position: relative;

		.drop-down {
			width: 128px;
			height: 40px;
			line-height: 40px;
			text-align: center;
			margin: 0 auto;
			color: #000000;
			background: #FFFFFF;
			position: relative;
			float: right;
			right: 20px;
			box-sizing: border-box;
			&:hover ul { opacity: 1; margin: 20px 0 0 0; }

			span {
				display: block;
				width: 100%;
				height: 100%;
			}
			ul {
				list-style-type: none;
				position: absolute;
				background: inherit;
				padding: 0;
				margin: 10px 0 0 0;
				opacity: 0;
				width: 100%;
				transition: all .5s ease;

				li:hover { background: #4f7f9b; color: #ffffff; }
			}
		}
	}
	#sec-2 .sec-2-content {
		padding-top: 80px;

		.cell {
			display: flex;
			align-items: center;
			justify-content: center;
		}
		.sec-2-list {
			display: flex;
			flex-direction: column;
			ul {
				display: flex;
				margin: 0 auto;
				flex-direction: row;
				border-bottom: 1px solid #ffffff;
				&:first-child { border-top: 1px solid #ffffff; }
				&:nth-child(2n) { background: rgba(43, 121, 179, .3); }
				span {
					width: 50px;
					@extend .cell;
					border: 1px solid #ffffff;
					border-width: 0 1px;
				}
				li {
					width: 180px;
					height: 30px;
					text-align:center;
					padding: 5px 10px;
					vertical-align: middle;
					box-sizing: border-box;
					box-shadow: 0px 0px 0px 1px #ffffff;
					&:hover { background: #ffffff; color: #000000; }
					@extend .cell;
				}
			}
		}
	}
	#sec-3 {}
	#sec-4 {
		h2 { font-size: 40px;  padding: 30px; }
	}
	#chart {
		width: 100%;
		height: 400px;
	}
</style>
