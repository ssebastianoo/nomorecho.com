<script lang="ts">
    import song1 from '$lib/assets/songs/sitting-next-to-you.opus';
    import song2 from '$lib/assets/songs/she-who-escapes.opus';
</script>

<div class="flex justify-center items-center flex-1 relative">
	<a class="hover:underline top-0 left-0 absolute" href="/">~home</a>
	<div class="text-left flex flex-col">
		<p class="text-lg mb-2">demos</p>

		<div>
			<a
				class="hover:underline"
				href="/songs/sitting-next-to-you.mp3"
				download="sitting-next-to-you.mp3">~sitting-next-to-you.mp3</a
			>

			<audio controls class="w-full my-4">
				<source src={song1} type="audio/ogg; codecs=opus" />
				Your browser does not support the audio element.
			</audio>
		</div>

		<div>
			<a class="hover:underline" href="/songs/she-who-escapes.mp3" download="she-who-escapes.mp3"
				>~she-who-escapes.mp3</a
			>

			<audio controls class="w-full my-4">
				<source src={song2} type="audio/ogg; codecs=opus" />
				Your browser does not support the audio element.
			</audio>
		</div>
	</div>
</div>
