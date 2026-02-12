<script>

    import { Command, Child} from '@tauri-apps/plugin-shell'
    import {downloadDir, extname, tempDir, basename, dirname} from '@tauri-apps/api/path'
    import { load, Store } from '@tauri-apps/plugin-store';
    import { open } from '@tauri-apps/plugin-dialog';
    import { writeFiles } from "tauri-plugin-clipboard-x-api"
    import { getCurrentWebview } from '@tauri-apps/api/webview';
    import { sendNotification } from '@tauri-apps/plugin-notification';
    import stringArgv from 'string-argv'
    import {stat, writeFile, remove} from "@tauri-apps/plugin-fs"
    import {onDestroy, onMount} from "svelte"

    const videoExtensions = [
        "mp4", "mov", "mkv", "webm", "avi",
        "m2ts", "mts", "mxf", "flv", "vob",
        "wmv", "3gp", "3g2", "mpg", "mpeg",
        "m4v", "ogv", "asf", "rm", "rmvb"
    ];

    const webview = getCurrentWebview();

    /** @type {string[]} */
    const files = []


    /** @callback unlistenfn
     *  @returns {void}
     */
     /**  @type {unlistenfn[]} */
    const listeners = []

    /** @type {Child[]} */
    const processes = []

    let compression_tasks = $state(0)
    let download_tasks = $state(0)

    let download_dir = ""
    let temp_dir = ""

    let notifications = $state(true)
    let compressed_to_clipboard = $state(true)

    /** @type {Store} */
    let store

    /* Todo
    * Better error handling & catching
    * Develop a way for the client log detailed errors to a server?
    */

    onMount(async () => {
        download_dir = await downloadDir()
        temp_dir = await tempDir()

        store = await load('settings.json');

        const notifications_store_value = await store.get("notifications")
        if (notifications_store_value !== undefined) {
            notifications = notifications_store_value
        }

        const compressed_to_clipboard_value = await store.get("compressed_to_clipboard")
        if (compressed_to_clipboard_value !== undefined) {
            compressed_to_clipboard = compressed_to_clipboard_value
        }

        console.log(download_dir)
        console.log(temp_dir)
    })

    function notify(input) {
        if (notifications) {
            sendNotification(input);
        }
    }

    class RunCommandOutput {
        /**
         * @param {number} code
         * @param {string} stdout
         * @param {string} stderr
         */
        constructor(code, stdout, stderr) {
            this.code = code
            this.stdout = stdout
            this.stderr = stderr
        }
    }

    async function run_command(name, args_string) {
        return new Promise(async (resolve, reject) => {

            name = `binaries/${name}`

            const command = Command.sidecar(name, stringArgv(args_string))

            let stderr = ""
            let stdout = ""

            command.on('error', error => {
                console.error(`Command failed: ${error}`);
            });

            command.stdout.on('data', line => {
                stdout += line
            });

            command.stderr.on('data', line => {
                stderr += line
            });

            // 1. Set up listeners before spawning
            command.on('close', data => {
                resolve(new RunCommandOutput(data.code, stdout, stderr))
            });

            processes.push(await command.spawn())
        })

    }

    async function compress_file(path) {

        /* Todo
        * Find a way to detect the overall "sharpness" and selectively apply gblur to avoid adding extra blur
        * For files that dont have audio, or audio size prediction fails remove audio with -an
        * Experiment with Region of interest for long files
        * Experiment with snapping fps to common rates (21fps, 24fps) and checking its effects on quality
        * Experiment replacing Ffmpeg with SvtAv1EncApp or Av1an for faster encoding and higher quality
        * Find a way to add back variable frame rates without destroying quality due to SVT's RDO (Rate Distortion Optimization)
        */

        console.log(path)
        if(!path){
            return
        }
        if(path instanceof Array) {
            path = path[0]
        }

        const fileExt = await extname(path)
        const baseName = await basename(path)
        const fileName = baseName.substring(0,baseName.length-(fileExt.length+1))
        const filePath = await dirname(path)

        if (!videoExtensions.includes(fileExt.toLowerCase())) {
            sendNotification({
                title: 'That is not a valid file format',
            });
            return
        }

        const log_file = `${temp_dir}\\log${Date.now()}`
        const outPath = `${filePath}\\${fileName}-amt.mp4`

        const fileInfo = await stat(path);
        console.log(fileInfo)
        if (fileInfo.size < 10485760) {
            if (compressed_to_clipboard) {
                await writeFiles([filePath])
                notify({
                    title: 'File Already Below 10MB',
                    body: `Copied it to your clipboard.`,
                });
            } else {
                notify({
                    title: 'File Already Below 10MB',
                });
            }
            return
        }

        await writeFile(outPath, new Uint8Array([]))
        files.push(outPath)

        compression_tasks = compression_tasks + 1

        console.log(fileName)
        console.log(filePath)

        let target_size = 9.85
        let video_resolution = 900
        let video_fps = 23
        let audio_bitrate = 40000
        let pass_1_preset = 9
        let pass_2_preset = 5
        let extra_audio_flags = ""

        const ffprobe_command = `-v error -show_entries format=duration -of default=noprint_wrappers=1:nokey=1 "${path}"`
        const ffprobe_output = await run_command('ffprobe', ffprobe_command)
        console.log(ffprobe_output)
        const duration_in_seconds = parseFloat(ffprobe_output.stdout.trim())
        console.log(duration_in_seconds)

        if (duration_in_seconds > 300) {
            pass_1_preset = 5
            pass_2_preset = 4
            video_resolution = 640
            video_fps = 22
            audio_bitrate = 22000
            extra_audio_flags = `-ac 1`
        }

        if (duration_in_seconds > 500) {
            pass_1_preset = 4
            pass_2_preset = 3
            video_resolution = 640
            video_fps = 18
            audio_bitrate = 18000
            extra_audio_flags = `-ac 1 -application voip`
        }

        let audio_size = 0.3
        // render audio to figure out how many MB's it takes up
        try {
            const audio_outpath = `${temp_dir}${Date.now()}.opus`
            const render_audio_command = `-i "${path}" -c:a libopus -b:a ${audio_bitrate} ${extra_audio_flags} -y "${audio_outpath}"`
            const audio_render_output = await run_command('ffmpeg', render_audio_command)
            audio_size = ((await stat(audio_outpath)).size / 1048576)
            await remove(audio_outpath)
            console.log(audio_outpath, audio_size)
        } catch {

        }

        const intermediate_out_path = `${temp_dir}${Date.now()}.mp4`
        files.push(intermediate_out_path)


        try {

            // gblur exists to prevent an oversharp aliased look on discord with clean videos above 1080p.
            const video_filter = `"fps=${video_fps},atadenoise,zscale=-2:'trunc(if(gt(ih,${video_resolution}),${video_resolution},ih)/2)*2':f=spline36:d=error_diffusion,hqdn3d=4:4:2:2,gblur=sigma=0.81:steps=1:enable='gte(h,900)',fspp=quality=4"`

            const intermediate_pass_command = ` -i "${path}" -c:v libx264 -preset ultrafast -c:a copy -crf 0 -vf ${video_filter} -threads 0 "${intermediate_out_path}"`
            const intermediate_pass_output = await run_command('ffmpeg', intermediate_pass_command)
            console.log(intermediate_pass_output)

            const get_frame_count_after_mpdecimate_command = `-i "${intermediate_out_path}" -hide_banner -nostdin -an -sn -dn -map 0:v:0 -f null NUL`
            const frame_count_mpdecimate_out = await run_command('ffmpeg', get_frame_count_after_mpdecimate_command)
            const matches = [...await frame_count_mpdecimate_out.stderr.matchAll(/frame=\s*(\d+)/g)];
            const total_frames = matches[matches.length-1][1]


            const video_bitrate = Math.round(((((target_size - audio_size) * 8 * 1000000)/total_frames) * (total_frames / duration_in_seconds)))
            console.log(`vbr: ${video_bitrate}`)

            const svtav1_params = `"tune=0:film-grain-denoise=0:enable-overlays=1:enable-qm=1:keyint=900:scd=1:rc=1:rc-lookahead=120:hierarchical-levels=5"`

            const first_pass_command = ` -i "${intermediate_out_path}" -c:v libsvtav1 -row-mt 1 -preset ${pass_1_preset} -svtav1-params ${svtav1_params} -pix_fmt yuv420p10le -threads 0 -movflags +faststart -b:v ${video_bitrate} -pass 1 -passlogfile ${log_file} -an -f null NUL`
            const first_pass_output = await run_command('ffmpeg', first_pass_command)
            console.log(first_pass_output)

            files.push(log_file)

            const second_pass_command = ` -i "${intermediate_out_path}" -c:v libsvtav1 -preset ${pass_2_preset} -svtav1-params ${svtav1_params} -pix_fmt yuv420p10le -movflags +faststart -threads 0 -b:v ${video_bitrate} -c:a libopus -b:a ${audio_bitrate} ${extra_audio_flags} -pass 2 -passlogfile ${log_file}  -y "${outPath}"`
            const second_pass_output = await run_command('ffmpeg', second_pass_command)
            console.log(second_pass_output)

            const index = files.indexOf(outPath)
            if (index !== -1) {
                files.splice(index, 1)
            }
        } catch {
            console.log("err has occured")
        }


        setTimeout(async ()=>{
            await remove(intermediate_out_path)
            const index = files.indexOf(intermediate_out_path)
            if (index !== -1) {
                files.splice(index, 1)
            }
        },2000)


        if (compressed_to_clipboard) {
            await writeFiles([outPath])
            notify({
                title: 'File Compressed!',
                body: `Copied ${fileName} to the clipboard.`,
            });
        } else {
            notify({
                title: 'File Compressed!',
                body: `${fileName}`,
            });
        }

        compression_tasks = compression_tasks - 1
    }

    let url_input = $state()

    async function download() {

        /* Todo
        * Find a way to extract file names ahead of time
        * Add and option to copy outputs to the clipboard
        * Add an option to download audio only
        * Catch errors
        */

        download_tasks += 1
        await run_command('yt-dlp', `-U`)
        await run_command('yt-dlp', `-o "${download_dir}\\%(title)s.%(ext)s" --no-mtime ${url_input}`)
        notify({
            title: 'File Downloaded!',
            body: `It's now located in your downloads directory.`,
        });
        download_tasks -= 1
    }

    async function onClick(event) {
        if(event.button === 0) {
            const selected = await open({
                multiple: false,
                filters: [{
                    name: 'Image',
                    extensions: videoExtensions,
                }]
            })
            await compress_file(selected)
        }
    }

    async function around_settings_clicked() {
        setTimeout(async ()=>{
            let tasks = []
            tasks.push(store.set("notifications", notifications))
            tasks.push(store.set("compressed_to_clipboard", compressed_to_clipboard))
            console.log("notifications ", notifications)
            console.log("compressed_to_clipboard", compressed_to_clipboard)
            await Promise.all(tasks)
            await store.save()
        },200)
    }

    (async () => {
        listeners.push(await webview.onDragDropEvent((event) => {
            if (event.payload.type === 'drop') {
                compress_file(event.payload.paths)
            }
        }))
    })()

    async function cleanup() {

        for (const unlisten of listeners) {
            unlisten()
        }


        let tasks = []

        for (const process of processes) {
            tasks.push(process.kill())
        }

        await Promise.all(tasks)

        tasks = []

        for (const filepath of files) {
            tasks.push(remove(filepath))
        }

        await Promise.all(tasks)
    }

    webview.listen('tauri://close-requested', () => {
        cleanup()
    }).then( event => {
        listeners.push(event)
    })

    onDestroy(()=>{
        for (const unlisten of listeners) {
            unlisten()
        }
    })


</script>


<div class="p-3 pb-0 text-[#f6f6f6] flex flex-col items-center justify-center w-[100vw] h-[100vh] bg-[#2f2f2f]">

    <div class="flex flex-col items-center justify-center w-full h-full gap-3">
        <div class="flex flex-row w-full gap-2">
            {#if download_tasks > 0}
                <div class="relative flex justify-center items-center p-0.5">
                    <div
                            class="animate-spin rounded-full w-[8vh] h-[8vh] border-4 border-t-transparent"
                            role="status"
                            aria-label="loading"
                    ></div>
                    <h1 class="absolute text-sm flex flex-row">{download_tasks}<span class="animate-pulse rounded-full">.</span>
                        <span class="animate-pulse rounded-full [animation-delay:-0.2s]">.</span>
                        <span class="animate-pulse rounded-full [animation-delay:-0.4s]">.</span></h1>
                </div>
            {/if}
            <input bind:value={url_input} class="w-full h-full rounded-sm border-2 border-neutral-500 p-2" placeholder="Paste a URL to download" type="text">
            <button onclick={download} class="bg-neutral-500 hover:bg-neutral-600 p-2 rounded-sm">
                <svg class="w-6 h-6 text-gray-800 dark:text-white" aria-hidden="true" xmlns="http://www.w3.org/2000/svg" width="24" height="24" fill="none" viewBox="0 0 24 24">
                    <path stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 13V4M7 14H5a1 1 0 0 0-1 1v4a1 1 0 0 0 1 1h14a1 1 0 0 0 1-1v-4a1 1 0 0 0-1-1h-2m-1-5-4 5-4-5m9 8h.01"/>
                </svg>
            </button>
        </div>
        <div role="button" tabindex="0" onpointerdown={onClick} class="flex select-none cursor-pointer border-2 rounded-sm flex-col items-center pt-2 border-t-2 border-neutral-500 justify-center w-full h-full">
            <h1>
                Drag or Select a File to be compressed
            </h1>
            {#if compression_tasks > 0}
                <div class="relative flex justify-center items-center p-5">
                    <div
                            class="animate-spin rounded-full w-[10vh] h-[10vh] border-4 border-t-transparent"
                            role="status"
                            aria-label="loading"
                    ></div>
                    <h1 class="absolute text-sm flex flex-row">{compression_tasks}<span class="animate-pulse rounded-full">.</span>
                        <span class="animate-pulse rounded-full [animation-delay:-0.2s]">.</span>
                        <span class="animate-pulse rounded-full [animation-delay:-0.4s]">.</span></h1>
                </div>
            {/if}
        </div>
        <div onpointerdown={around_settings_clicked} class="flex flex-row flex-wrap w-full">
            <label class="flex flex-row text-sm select-none gap-1.5 p-2 pt-1.5 hover:bg-neutral-800 rounded-sm" >
                <h1>Copy Compressed files to clipboard</h1>
                <input bind:checked={compressed_to_clipboard} type="checkbox">
            </label>
            <label class="flex flex-row text-sm select-none gap-1.5 p-2 pt-1.5 hover:bg-neutral-800 rounded-sm" >
                <h1>Notifications</h1>
                <input bind:checked={notifications} type="checkbox">
            </label>
        </div>
    </div>
</div>