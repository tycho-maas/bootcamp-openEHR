<script lang="ts">
    import "medblocks-ui";
    import "medblocks-ui/dist/styles.js";
    import webtemplate from "./vital_signs_tycho_maas.v1.json";
    let ehrId = "ceb6ca7d-3a90-4943-a473-760cb7437d48";
    let openEHRRestEndpoint = "https://openehr-bootcamp.medblocks.com/ehrbase/rest/openehr/v1";
    let form: any

      async function submitVitals() {
        const composition = form.export()
        console.log(composition)
        let response = await fetch (`${openEHRRestEndpoint}/ehr/${ehrId}/composition?templateId=${webtemplate.templateId}` , {
          method: "POST",
          body: JSON.stringify(composition),
          headers: {
            "Content-Type": "application/openehr.wt.flat.schema+json",
            "Accept": "application/openehr.wt.flat.schema+json",
            "Prefer": " return=representation"

          }
        })
        let data = await response.json()
        console.log(data)
}

</script>

<div class="flex w-full min-h-screen">
  <!-- Left: Form Section (white background) -->
  <div class="w-1/2 p-4 bg-white flex flex-col">
    <h2 class="text-2xl font-semibold mb-4">Nursing Form</h2>
    <mb-auto-form bind:this={form} webTemplate={webtemplate} class="flex-1"></mb-auto-form>
    <div class="mt-4 text-center">
      <button on:click={submitVitals} class="bg-blue-500 text-white text-lg font-semibold py-3 px-8 rounded-full shadow-md hover:bg-blue-600 transition duration-300">
        Submit
      </button>
    </div>
  </div>

  <!-- Right: Previous Entries Section (gray background) -->
  <div class="w-1/2 p-4 bg-gray-100 flex flex-col">
    <h2 class="text-2xl font-semibold mb-4">Previous Entries</h2>
    <div class="flex-1">
      <!-- Add previous entries here -->
    </div>
  </div>
</div>
