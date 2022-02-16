<script type="ts">
	import { onMount } from 'svelte';
	import { browser } from '$app/env';
	import type { ContractMethods } from 'near-api-js/lib/contract';
	import type { AccountBalance } from 'near-api-js/lib/account';
	import type { Near, Account, WalletConnection, Contract, keyStores } from 'near-api-js';
	import { utils } from '$lib/@near-api-js';
	window['process'] = {};
	function sleep(ms) {
		return new Promise((resolve) => setTimeout(resolve, ms));
	}
	const nearAPI = window['nearApi'];
	const CONTRACT_NAME = 'app_2.spin_swap.testnet';

	const CONTRACT_METHODS: ContractMethods = {
		changeMethods: [
			'ask',
			'bid',
			'drop_order',
			'deposit_near',
			'withdraw',
			'drop_all_orders',
			'batch_ops'
		],
		viewMethods: [
			'markets',
			'view_market',
			'get_orders',
			'get_order_by_id',
			'spin_trade_get_balance',
			'get_all_deposits'
		]
	};

	let near: Near;
	let account: Account;
	let wallet: WalletConnection;
	let signedIn = false;
	let balances: AccountBalance;
	let deposits;
	let baseContract: Contract;
	let pk;
	const keyStore = new nearAPI.keyStores.BrowserLocalStorageKeyStore(localStorage, 'spin_mm:');

	const config = {
		networkId: 'testnet',
		nodeUrl: 'https://rpc.testnet.near.org',
		walletUrl: 'https://wallet.testnet.near.org',
		helperUrl: 'https://helper.testnet.near.org',
		explorerUrl: 'https://explorer.testnet.near.org',
		headers: {},
		keyStore
	};

	onMount(async () => {
		let keyPair = nearAPI.KeyPair.fromRandom('ed25519');
		pk = keyPair.getPublicKey().toString();
		keyStore.setKey(config.networkId, pk, keyPair);
		// const kkk = await keyStore.getKey(config.networkId, 'chedrik.testnet');
		// console.log(kkk.getPublicKey().toString());
		// console.log(kkk.getPublicKey().toString());
		// config.keyStore = keyStore;
		near = await nearAPI.connect(config);
		wallet = new nearAPI.WalletConnection(near, 'spin');
		account = await near.account(wallet.getAccountId());
		// const keys = await account.getAccessKeys();
		// console.log(keys);
		signedIn = wallet.isSignedIn();
		balances = await wallet.account().getAccountBalance();
		baseContract = new nearAPI.Contract(wallet.account(), CONTRACT_NAME, CONTRACT_METHODS);

		// console.log(baseContract, pk);
	});

	async function addAccessKey() {
		let newKeyPair = nearAPI.KeyPair.fromRandom('ed25519');
		let keystore = new nearAPI.keyStores.BrowserLocalStorageKeyStore(localStorage, 'spin_mm:');
		keystore.setKey('testnet', wallet.account().accountId, newKeyPair);

		const actions = [
			await nearAPI.transactions.addKey(
				newKeyPair.publicKey,
				nearAPI.transactions.functionCallAccessKey(
					CONTRACT_NAME,
					[
						'ask',
						'bid',
						'drop_order',
						'deposit_near',
						'withdraw',
						'drop_all_orders',
						'batch_ops',
						'markets',
						'view_market',
						'get_orders',
						'get_order_by_id',
						'spin_trade_get_balance',
						'get_all_deposits'
					],
					utils.format.parseNearAmount('0')
				)
			)
			//2nd Action would go here if we wanted to do more things in this call
		];

		// this signs and sends the transaction
		await wallet.account().signAndSendTransaction(wallet.getAccountId(), actions);
	}

	function signIn() {
		wallet.requestSignIn(
			{
				contractId: CONTRACT_NAME,
				methodNames: [
					'ask',
					'bid',
					'drop_order',
					'deposit_near',
					'withdraw',
					'drop_all_orders',
					'batch_ops',
					'markets',
					'view_market',
					'get_orders',
					'get_order_by_id',
					'spin_trade_get_balance',
					'get_all_deposits'
				]
			},
			'Spin'
		);
	}
	function signOut() {
		wallet.signOut();
		signedIn = wallet.isSignedIn();
	}

	async function getMarketList() {
		if (browser) {
			const marketsRAW = await account.viewFunction(CONTRACT_NAME, 'markets');
			console.dir(marketsRAW);
		}
	}

	async function depositNear() {
		if (browser) {
			await baseContract['deposit_near'](
				{},
				'50000000000000',
				nearAPI.utils.format.parseNearAmount('1')
			);
		}
	}
	function generateRandomNumber(min: number, max: number): number {
		return Math.random() * (max - min) + min;
	}

	async function createBuyOrder(market_id: number, price: string, size: string) {
		await baseContract['ask'](
			{
				market_id: 8,
				price: utils.format.parseNearAmount(price),
				quantity: utils.format.parseNearAmount(size),
				ttl: 60 * 60 * 24 * 7,
				market_order: false
			},
			'50000000000000'
		);
	}

	async function dropGrid() {
		const orders = await getOrders();
		const _askOrders = orders.filter((o) => o.o_type === 'Ask').map((o) => o.id);
		const _bidOrders = orders.filter((o) => o.o_type === 'Bid').map((o) => o.id);
		console.log(_askOrders, _bidOrders);
		if (_askOrders.length) {
			await baseContract['batch_ops'](
				{
					ops: [
						{
							market_id: 8,
							drop: _askOrders,
							place: []
						}
					]
				},
				'300000000000000'
			);
		}
		await sleep(1000);
		if (_bidOrders.length) {
			await baseContract['batch_ops'](
				{
					ops: [
						{
							market_id: 8,
							drop: _bidOrders,
							place: []
						}
					]
				},
				'300000000000000'
			);
		}
	}
	async function placeGrid() {
		let _bids = BIDS.map((b) => ({
			market_id: 8,
			ttl: 604800, // 1 week
			market_order: false,
			order_type: 'Bid',
			price: utils.format.parseNearAmount(b.price.toFixed(4)),
			quantity: utils.format.parseNearAmount(b.size.toFixed(4))
		}));
		let _asks = ASKS.map((a) => ({
			market_id: 8,
			ttl: 604800, // 1 week
			market_order: false,
			order_type: 'Ask',
			price: utils.format.parseNearAmount(a.price.toFixed(4)),
			quantity: utils.format.parseNearAmount(a.size.toFixed(4))
		}));

		await baseContract['batch_ops'](
			{
				ops: [
					{
						market_id: 8,
						drop: [],
						place: [..._bids]
					}
				]
			},
			'300000000000000'
		);
		await sleep(5000);
		await baseContract['batch_ops'](
			{
				ops: [
					{
						market_id: 8,
						drop: [],
						place: [..._asks]
					}
				]
			},
			'300000000000000'
		);
	}

	async function getOrders() {
		const orders = await baseContract['get_orders']({
			market_id: 8,
			account_id: 'chedrik.testnet'
		});
		return orders;
	}

	async function randomizer() {
		let price = generateRandomNumber(11.7, 12);
		let size = generateRandomNumber(0.25, 2);

		await createBuyOrder(8, price, size).then(() => {
			console.info(`BUY Order @ ${price} — ${size}`);
			setTimeout(async () => await randomizer(), 10000);
		});
	}

	let BINANCE_PRICE = 0;
	async function binancePrice() {
		await fetch('https://api.binance.com/api/v3/ticker/price?symbol=NEARUSDT')
			.then((r) => r.json())
			.then((r) => {
				BINANCE_PRICE = parseFloat(r.price);
			});
	}
	interface PriceNode {
		price: number;
		size: number;
	}
	let ASKS: PriceNode[] = [];
	let BIDS: PriceNode[] = [];

	async function buildGrid(
		spread: number = 1,
		maxSizeBid: number,
		maxSizeAsk: number,
		ordersNumber: number = 15
	) {
		await binancePrice();
		let currentPrice = BINANCE_PRICE;
		let _bids = [];
		let _asks = [];
		for (let index = 0; index < ordersNumber; index++) {
			let _bPrice = generateRandomNumber(currentPrice - spread, currentPrice);
			let _bSize = generateRandomNumber(0.1, maxSizeBid / ordersNumber);
			let _aPrice = generateRandomNumber(currentPrice + spread, currentPrice);
			let _aSize = generateRandomNumber(0.1, maxSizeAsk / ordersNumber);

			_bids.push({
				price: _bPrice,
				size: _bSize / _bPrice
			});
			_asks.push({
				price: _aPrice,
				size: _aSize / _aPrice
			});
		}

		BIDS = _bids.sort((a, b) => {
			return b.price - a.price;
		});

		ASKS = _asks.sort((a, b) => {
			return a.price - b.price;
		});
	}

	async function autoPlace() {
		await dropGrid();
		await buildGrid(0.25, 500000, 500, 15);
		await placeGrid();
		await sleep(60 * 1000);
		await autoPlace();
	}
</script>

<h1>Welcome to SPIN</h1>
<h2>Auth</h2>
{#if !signedIn}
	<button on:click={() => signIn()}>Sign in to SPIN</button>
{:else}
	wallet id:
	<strong>{wallet?.getAccountId()}</strong>
{/if}
<button on:click={() => signOut()}>Sign out</button>

<button on:click={async () => await addAccessKey()}>Increase Allowlance</button>

<h2>RANDOMIZER BLYAT</h2>
<p>Binance price: {BINANCE_PRICE}</p>
<table>
	<thead>
		<th>BID total</th>
		<th style="color:green;">BID Size</th>
		<th style="color:green;">BID Price</th>
		<th style="color:tomato;">ASK Pize</th>
		<th style="color:tomato;">ASK Size</th>
		<th>ask total</th>
	</thead>
	<tbody>
		{#each Array(BIDS.length) as _, i}
			<tr>
				<td>{(BIDS[i]?.size * BIDS[i]?.price).toFixed(4)}</td>
				<td>{BIDS[i]?.size.toFixed(4)}</td>
				<td>{BIDS[i]?.price.toFixed(4)}</td>
				<td>{ASKS[i]?.price.toFixed(4)}</td>
				<td>{ASKS[i]?.size.toFixed(4)}</td>
				<td>{(ASKS[i]?.size * ASKS[i]?.price).toFixed(4)}</td>
			</tr>
		{/each}
	</tbody>
</table>
<!-- <button on:click={async () => await randomizer()}>Place Buy order</button> -->
<button on:click={async () => await buildGrid(0.25, 500000, 500, 15)}>Build MM grid</button>
<button on:click={async () => await placeGrid()}>Place MM grid</button>
<button on:click={async () => await autoPlace()}>Auto Place</button>
<button on:click={async () => await dropGrid()}>Drop MM grid</button>
<button on:click={async () => await getOrders().then((o) => console.info(o))}>Get Orders</button>

<h2>Balances</h2>
{#if balances}
	<p>Total: {nearAPI.utils.format.formatNearAmount(balances.total)} NEAR</p>
	<p>Staked: {nearAPI.utils.format.formatNearAmount(balances.staked)} NEAR</p>
	<p>Available: {nearAPI.utils.format.formatNearAmount(balances.available)} NEAR</p>
{/if}

<h2>Deposits</h2>
deposit_near
<button on:click={async () => depositNear()}>Deposit 1 Near</button>

<h2>Markets</h2>
<button on:click={async () => getMarketList()}>Get Market List</button>

<style>
	table,
	th,
	td {
		border: 1px solid black;
		border-collapse: collapse;
		font-family: monospace;
	}
</style>
