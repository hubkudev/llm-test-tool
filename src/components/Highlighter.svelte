<script lang="ts">
    import Highlight, { LineNumbers } from "svelte-highlight";
    import json from "svelte-highlight/languages/json";
    import style from "svelte-highlight/styles/night-owl";

    interface Props {
        // Code that will be Highlighted by the library
        code: string;
    }

    let { code }: Props = $props();
</script>

<svelte:head>
    {@html style}
</svelte:head>

<Highlight language={json} {code} let:highlighted>
    <LineNumbers hideBorder {highlighted} />
</Highlight>
