<script>
    import '../app.css'
    let {children} = $props()

    import {onMount} from "svelte"
    import {getCurrentWindow} from "@tauri-apps/api/window"

    onMount(async () => {
    })

</script>


{@render children()}