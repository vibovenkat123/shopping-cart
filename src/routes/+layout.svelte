<script lang="ts">
	import '../app.css';
	import { company } from '../fake';
	import { cart } from '../stores/cart';
	import { total } from '../stores/cart';
	import Login from '../components/Login.svelte';
	import Profile from '../components/Profile.svelte';
	import { signInWithPopup } from 'firebase/auth';
	import { auth, provider, user, db } from '../firebase';
	import { collection, query, where, getDocs, addDoc, deleteDoc } from 'firebase/firestore';

	let homeActive: string;
	let productsActive: string;
	if (typeof location !== 'undefined') {
		if (location.pathname == '/Shop') {
			productsActive = 'underline';
			homeActive = 'none';
		} else if (location.pathname == '/') {
			productsActive = 'none';
			homeActive = 'underline';
		} else {
			productsActive = 'none';
			homeActive = 'none';
		}
	}
	async function addItem(id: number, price: number) {
		let index = 1;
		for (const i in $cart) {
			if ($cart[i].id == id) {
				cart.update((n) => {
					n[i].quantity++;
					return n;
				});
				index = parseInt(i);
			}
		}
		total.update((n) => n + price);
		const q = query(collection(db, 'carts'), where('data.id', '==', id));
		const querySnapshot = await getDocs(q);
		querySnapshot.forEach((doc) => {
			deleteDoc(doc.ref);
		});
		addDoc(collection(db, 'carts'), { data: $cart[index], uid: $user!.uid, total: $total });
	}
	let modal: HTMLDialogElement;
	async function showCart() {
		let res = await getTheUid();
		cart.set(res);
		modal.showModal();
	}
	function closeCart() {
		modal.close();
	}
	async function getTheUid() {
		const q = query(collection(db, 'carts'), where('uid', '==', $user!.uid));
		const querySnapshot = await getDocs(q);
		let temp: { id: number; quantity: number }[] = [];
		let totalTemp = 0;
		querySnapshot.forEach((doc) => {
			temp.push(doc.data().data);
			if (doc.data().total > totalTemp) {
				totalTemp = doc.data().total;
			}
		});
		total.set(totalTemp);
		return temp;
	}
	async function removeItem(id: number, price: number) {
		for (const i in $cart) {
			if ($cart[i].id == id) {
				if ($cart[i].quantity <= 1) {
					cart.update((n) => {
						n.splice(parseInt(i), 1);
						return n;
					});
					total.update((n) => n - price);
					const q = query(collection(db, 'carts'), where('data.id', '==', id));
					const querySnapshot = await getDocs(q);
					querySnapshot.forEach((doc) => {
						deleteDoc(doc.ref);
					});
				} else {
					cart.update((n) => {
						n[i].quantity--;
						return n;
					});
					total.update((n) => n - price);
					if (isNaN($total)) {
						console.log('yo');
						total.set(Infinity);
						console.log($total);
					}
					const q = query(collection(db, 'carts'), where('data.id', '==', id));
					const querySnapshot = await getDocs(q);
					querySnapshot.forEach((doc) => {
						deleteDoc(doc.ref);
					});
					addDoc(collection(db, 'carts'), {
						data: $cart[parseInt(i)],
						uid: $user!.uid,
						total: $total
					});
					break;
				}
			}
		}
	}
	let companyName = company.name;
</script>

<main>
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" />
	<link
		rel="stylesheet"
		href="https://fonts.googleapis.com/css2?family=Material+Symbols+Outlined:opsz,wght,FILL,GRAD@48,400,0,0"
	/>
	<link
		href="https://fonts.googleapis.com/css2?family=Roboto+Mono:ital,wght@0,300;0,500;0,700;1,400&display=swap"
		rel="stylesheet"
	/>
	<link
		rel="stylesheet"
		href="https://fonts.googleapis.com/css2?family=Material+Symbols+Outlined:opsz,wght,FILL,GRAD@48,400,0,0"
	/>
	<header class="bg-nord1 flex justify-center h-24">
		<div class="flex justify-center items-center p-4 w-3/4 text-nord6 flex-col">
			<div>
				<a href="/" class="w-full">
					<h1 class="text-5xl font-RobotoM font-normal" aria-label="xoringer">
						{companyName}
					</h1>
				</a>
			</div>
			<nav class="w-full flex justify-between items-center gap-7 text-2xl">
				<a
					href="/"
					style="text-decoration: {homeActive}; text-underline-offset: 8px;"
					on:click={() => {
						homeActive = 'underline';
						productsActive = 'none';
					}}>Home</a
				>
				<a
					href="/Shop"
					on:click={() => {
						productsActive = 'underline';
						homeActive = 'none';
					}}
					style="text-decoration: {productsActive}; text-underline-offset: 8px;">Shop</a
				>
				{#if $user}
					<Profile name={$user.displayName} imageURL={$user.photoURL} />
				{:else}
					<button
						on:click={() => {
							signInWithPopup(auth, provider);
						}}>Login</button
					>
				{/if}
				<div>
					<button aria-label="cart" on:click={showCart}>
						<span
							class="material-symbols-outlined align-bottom"
							style="font-size: 28px;"
							aria-hidden="true"
						>
							shopping_cart
						</span>
					</button>
				</div>
			</nav>
		</div>
	</header>
	<div class="space wave-down" />
	<dialog class="bg-nord3 text-nord6 border-2 w-96 border-nord6" bind:this={modal}>
		<div class="overflow-auto p-2">
			<button on:click={closeCart}> <span class="material-symbols-outlined"> close </span></button>
			<Login {addItem} {removeItem} />
		</div>
	</dialog>
</main>
<slot />

<style scoped>
	.space {
		aspect-ratio: 960/95;
		width: 100%;
		background-repeat: no-repeat;
		background-position: center;
		background-size: cover;
	}
	.wave-down {
		position: relative;
		top: 0px;
		background-image: url('../images/waves-down.svg');
	}
</style>
