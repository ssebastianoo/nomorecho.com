<script lang="ts">
    import song1 from '$lib/assets/songs/sitting-next-to-you.opus';
    import song2 from '$lib/assets/songs/she-who-escapes.opus';
	import song3 from '$lib/assets/songs/i-will-not-change-my-mind.opus';
	import song4 from '$lib/assets/songs/y-u-8-me.opus';
	import song5 from '$lib/assets/songs/you-dont-care-about-us.opus';
</script>

<a class="hover:underline" href="/">~home</a>
<div class="flex justify-center items-center flex-1 mt-2">
	<div class="text-left flex flex-col">
		<p class="text-lg mb-2">demos</p>
		<div class="grid md:grid-cols-2 gap-x-4 gap-y-2">
			<div>
				<a
					class="hover:underline text-pink-400"
					href="/songs/sitting-next-to-you.mp3"
					download="sitting-next-to-you.mp3">~sitting-next-to-you.mp3</a
				>

				<audio controls class="w-full my-4">
					<source src={song1} type="audio/ogg; codecs=opus" />
					Your browser does not support the audio element.
				</audio>
			</div>

			<div>
				<a
					class="hover:underline text-pink-400"
					href="/songs/she-who-escapes.mp3"
					download="she-who-escapes.mp3">~she-who-escapes.mp3</a
				>

				<audio controls class="w-full my-4">
					<source src={song2} type="audio/ogg; codecs=opus" />
					Your browser does not support the audio element.
				</audio>
			</div>

			<div>
				<a
					class="hover:underline text-pink-400"
					href="/songs/i-will-not-change-my-mind.mp3"
					download="i-will-not-change-my-mind.mp3">~i-will-not-change-my-mind.mp3</a
				>

				<audio controls class="w-full my-4">
					<source src={song3} type="audio/ogg; codecs=opus" />
					Your browser does not support the audio element.
				</audio>
			</div>

			<div>
				<a class="hover:underline text-pink-400" href="/songs/y-u-8-me.mp3" download="y-u-8-me.mp3"
					>~y-u-8-me.mp3</a
				>

				<audio controls class="w-full my-4">
					<source src={song4} type="audio/ogg; codecs=opus" />
					Your browser does not support the audio element.
				</audio>
			</div>

			<div>
				<a
					class="hover:underline text-pink-400"
					href="/songs/you-dont-care-about-us.mp3"
					download="you-dont-care-about-us.mp3">~you-dont-care-about-us.mp3</a
				>

				<audio controls class="w-full my-4">
					<source src={song5} type="audio/ogg; codecs=opus" />
					Your browser does not support the audio element.
				</audio>
			</div>
		</div>
	</div>
</div>
