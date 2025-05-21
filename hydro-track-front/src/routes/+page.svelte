<script>
    import { goto } from '$app/navigation';
    let tipo = "usuario";
    let id = "";

  function entrar() {
    if (tipo === "usuario") {
      document.cookie = `deviceId=${id}`;
      goto("/dashboard");
    } else if (tipo === "admin") {
      goto(`/dashboard-admin/${id}`);
    }
  }
</script>

<div class="min-h-screen flex items-center justify-center">
  <div class="bg-gray-800 p-8 rounded-lg shadow-lg w-full max-w-md">
    <h1 class="text-3xl font-bold mb-6 text-center">Entrar</h1>
    <div class="mb-4">
      <label for="tipo-select" class="block mb-2">Tipo de usuário:</label>
      <select id="tipo-select" bind:value={tipo} class="w-full bg-gray-700 border border-gray-600 rounded px-3 py-2">
        <option value="usuario">Usuário Comum</option>
        <option value="admin">Administrador</option>
      </select>
    </div>
    <div class="mb-6">
      <label for="id-input" class="block mb-2">{tipo === "usuario" ? "ID do Dispositivo" : "ID do Admin"}:</label>
      <input id="id-input" type="text" bind:value={id} class="w-full bg-gray-700 border border-gray-600 rounded px-3 py-2" />
    </div>
    <button on:click={entrar} class="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 rounded">
      Entrar
    </button>
  </div>
</div>
