<script setup lang='ts'>
import Editable from './components/Editable.vue' 
</script>

<template>
	<div class="container">
		<Editable/>
	</div>
</template>

<style>
* {
	margin: 0;
	padding: 0;

	background-color: #18181b;
	color: white;
}

.container {
	width: 100vw;
	height: 100vh;

	display: flex;
	justify-content: center;
	align-items: center;
}
</style>

