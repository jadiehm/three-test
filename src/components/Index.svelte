<script>
	import { getContext } from "svelte";
	import Footer from "$components/Footer.svelte";
	import ScriptAnimation from "$components/ThreeStudio.svelte";
	import TextureHalftone from "$components/TextureHalftone.svelte";
	import TypwriterImage from "$components/TypwriterImage.svelte";

	// const copy = getContext("copy");
	// const data = getContext("data");
</script>

<svelte:boundary onerror={(e) => console.error(e)}>
	<TextureHalftone />
</svelte:boundary>
