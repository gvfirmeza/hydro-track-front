<script>
  import { onMount } from "svelte";
  import { goto } from "$app/navigation";
  import Header from "$lib/Header.svelte";

  onMount(() => {
    // Check if we're at the base dashboard-admin route
    // If so, redirect to the dynamic route with the admin ID
    const path = window.location.pathname;
    if (path === "/dashboard-admin") {
      // Get admin ID from URL query or redirect to login
      const urlParams = new URLSearchParams(window.location.search);
      const adminId = urlParams.get('adminId');
      
      if (adminId) {
        goto(`/dashboard-admin/${adminId}`);
      } else {
        goto('/');
      }
    }
  });
</script>

<div class="min-h-screen">
  <Header />
  <div class="max-w-[90%] m-auto p-6 space-y-8">
    <h1 class="text-4xl text-center font-bold mt-4 mb-10 break-words">
      Redirecionando...
    </h1>
  </div>
</div>
