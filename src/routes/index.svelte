<script type="ts">
	import { onMount } from 'svelte';
	import type { AccountBalance } from 'near-api-js/lib/account';
	import type { Near, WalletConnection } from 'near-api-js';
	import { formatNearAmount } from 'near-api-js/lib/utils/format';

	const nearAPI = window['nearApi'];

	let near: Near;
	let wallet: WalletConnection;
	let signedIn = false;
	let balances: AccountBalance;
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
		near = await nearAPI.connect(config);
		wallet = new nearAPI.WalletConnection(near, 'spin');
		signedIn = wallet.isSignedIn();
		balances = await wallet.account().getAccountBalance();
	});

	function signIn() {
		wallet.requestSignIn({}, 'Spin');
	}

	function signOut() {
		wallet.signOut();
		signedIn = wallet.isSignedIn();
	}
	async function functionCallExample() {
		await wallet.account().functionCall({
			contractId: '',
			methodName: '',
			args: {}
			// gas?: BN,
			// attachedDeposit?: BN,
		});
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

<h2>Example:</h2>
<button on:click={async () => await functionCallExample()}>Example Function Call</button>

<h2>Balances</h2>
{#if balances}
	<p>Total: {formatNearAmount(balances.total)} NEAR</p>
	<p>Staked: {formatNearAmount(balances.staked)} NEAR</p>
	<p>Available: {formatNearAmount(balances.available)} NEAR</p>
{/if}
