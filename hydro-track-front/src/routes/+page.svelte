<script>
    import { goto } from "$app/navigation";

    let deviceId = "";

    function entrar() {
        if (deviceId) {
            document.cookie = `deviceId=${deviceId}; path=/;`;
            goto("/dashboard");
        }
    }
</script>

<div class="min-h-screen flex flex-col items-center justify-center p-4">
    <h1 class="text-2xl font-bold mb-6">HydroTrack - Login</h1>
    <input
        type="text"
        placeholder="Digite seu Device ID"
        bind:value={deviceId}
        class="border p-2 rounded w-64 mb-4"
    />
    <button
        on:click={entrar}
        class="bg-blue-500 hover:bg-blue-600 text-white px-4 py-2 rounded"
    >
        Entrar
    </button>
</div>
