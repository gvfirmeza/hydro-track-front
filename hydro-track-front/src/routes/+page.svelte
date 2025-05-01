<script>
    import { goto } from "$app/navigation";
    import Header from "$lib/Header.svelte";

    let deviceId = "";

    function entrar() {
        if (deviceId) {
            document.cookie = `deviceId=${deviceId}; path=/;`;
            goto("/dashboard");
        }
    }
</script>

<div class="min-h-screen bg-gray-900 text-gray-200">
    <Header />
    <div class="flex flex-col items-center justify-center p-60">
        <h1 class="text-2xl font-bold mb-6">HydroTrack - Login</h1>
        <input
            type="text"
            placeholder="Digite seu Device ID"
            bind:value={deviceId}
            class="border border-gray-700 bg-gray-800 text-gray-200 p-2 rounded w-64 mb-4"
        />
        <button
            on:click={entrar}
            class="bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded"
        >
            Entrar
        </button>
    </div>
</div>
