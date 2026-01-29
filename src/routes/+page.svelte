<script lang="ts">
	import inverted from '$lib/assets/inverted.png';
</script>

<div class="flex justify-center items-center flex-1">
	<div class="text-left space-y-4">
		<img src={inverted} width="150" alt="nomorecho" class="mx-auto" />
		<div>
			<p class="text-2xl">nomorecho</p>
		</div>
		<div>
			<p class="mb-1 text-lg">listen_us</p>
			<div class="grid grid-cols-2 text-pink-400">
				<a
					class="hover:underline"
					target="_blank"
					href="https://open.spotify.com/artist/4dctRUcKxGPNXcTEk9mzTF">~spotify</a
				>
				<a
					class="hover:underline"
					target="_blank"
					href="https://music.apple.com/it/artist/nomorecho/1848701396">~apple_music</a
				>
				<a class="hover:underline" target="_blank" href="https://tidal.com/artist/68940367"
					>~tidal</a
				>
				<a class="hover:underline" target="_blank" href="https://www.deezer.com/en/artist/352878482"
					>~deezer</a
				>
			</div>
		</div>

		<div>
			<p class="mb-1 text-lg">follow_us</p>

			<div class="grid grid-cols-2 text-pink-400">
				<a class="hover:underline" target="_blank" href="https://www.instagram.com/nomorechoband"
					>@instagram</a
				>

				<a class="hover:underline" target="_blank" href="https://www.tiktok.com/@nomorecho"
					>@tiktok</a
				>
			</div>
		</div>

		<div>
			<p class="mb-1 text-lg">shh_!</p>
			<div class="grid grid-cols-1 text-pink-400">
				<a class="hover:underline" href="/demos">~👀</a>
			</div>
		</div>
	</div>
</div>
