<script lang="ts">
    import "medblocks-ui";
    import "medblocks-ui/dist/styles.js";
    import webtemplate from "./vital_signs_tycho_maas.v1.json";

    let ehrId = "ceb6ca7d-3a90-4943-a473-760cb7437d48";
    let openEHRRestEndpoint = "https://openehr-bootcamp.medblocks.com/ehrbase/rest/openehr/v1";
    let form: any;
    let vitals: { uid: string, timestamp: string }[] = [];
    let showToast = false;
    let toastMessage = "";
    let toastTimeout: any;
    let updatingVitals: string | null = null


    async function submitVitals() {
        // Show loading toast
        if (!updatingVitals) {
            toastMessage = "Submitting...";
            showToast = true;
            clearTimeout(toastTimeout);

            const composition = form.export();

            let response = await fetch(`${openEHRRestEndpoint}/ehr/${ehrId}/composition?templateId=${webtemplate.templateId}`, {
                method: "POST",
                body: JSON.stringify(composition),
                headers: {
                    "Content-Type": "application/openehr.wt.flat.schema+json",
                    "Accept": "application/openehr.wt.flat.schema+json",
                    "Prefer": "return=representation"
                }
            });

            let data = await response.json();

            if (response.ok) {
                toastMessage = "Submission successful!";
                form.import({}); // Clear form only after success
                loadVitals(); // Refresh list
            } else {
                toastMessage = "Submission failed.";
                console.error("POST failed:", data);
            }

            toastTimeout = setTimeout(() => showToast = false, 3000);
            console.log("Successfully submitted vitals.", data);
        } else {
            // For updating vitals - explicitly show toast at the beginning
            toastMessage = "Updating...";
            showToast = true;
            clearTimeout(toastTimeout);

            const compositionId = updatingVitals.split("::")[0]; // Ensure we split the composition ID
            const composition = form.export(); // Get the updated composition

            let response = await fetch(`${openEHRRestEndpoint}/ehr/${ehrId}/composition/${compositionId}?templateId=${webtemplate.templateId}`, {
                method: "PUT",
                body: JSON.stringify(composition),
                headers: {
                    "Content-Type": "application/openehr.wt.flat.schema+json",
                    "Accept": "application/openehr.wt.flat.schema+json",
                    "Prefer": "return=representation",
                    "If-Match": updatingVitals
                }
            });

            let data = await response.json();

            if (response.ok) {
                toastMessage = "Update successful!";
                showToast = true; // Ensure toast is visible
                loadVitals(); // Refresh list
                form.import({}); // Clear form after update
                updatingVitals = null; // Clear the editing state
            } else {
                toastMessage = "Update failed.";
                showToast = true; // Ensure toast is visible even on failure
                console.error("PUT failed:", data);
            }

            // Display the toast for 3 seconds after the update
            toastTimeout = setTimeout(() => showToast = false, 3000);
            console.log("Successfully updated vitals.", data);
        }
    }


    async function loadVitals() {
        let response = await fetch(`${openEHRRestEndpoint}/query/aql`, {
            method: "POST",
            body: JSON.stringify({
                q: `SELECT c / uid / value, c / context / start_time / value
                    FROM EHR e CONTAINS COMPOSITION c
                    WHERE c /archetype_details/template_id/ value = '${webtemplate.templateId}'
                      AND e/ehr_id/ value = '${ehrId}'`
            }),
            headers: {
                "Content-Type": "application/json"
            }
        });
        let data = await response.json();
        vitals = data.rows.map(([uid, timestamp]: [string, string]) => ({uid, timestamp}));
    }

    function formatDateTime(iso: string) {
        const dateObj = new Date(iso);
        const options: Intl.DateTimeFormatOptions = {
            weekday: 'long',
            year: 'numeric',
            month: 'long',
            day: 'numeric',
            hour: '2-digit',
            minute: '2-digit',
            second: '2-digit',
            hour12: true
        };
        return dateObj.toLocaleDateString('en-US', options);
    }

    async function editVital(vitalUid: string) {
        toastMessage = "Loading entry...";
        showToast = true;
        clearTimeout(toastTimeout);

        const response = await fetch(`${openEHRRestEndpoint}/ehr/${ehrId}/composition/${vitalUid}`, {
            headers: {
                "Accept": "application/openehr.wt.flat.schema+json",
            }
        });

        if (response.ok) {
            const data = await response.json();
            form.import(data);
            updatingVitals = vitalUid;
            toastMessage = "Entry loaded.";
            toastTimeout = setTimeout(() => showToast = false, 3000);
        } else {
            toastMessage = "Failed to load entry.";
            console.error("Fetch failed:", await response.text());
            toastTimeout = setTimeout(() => showToast = false, 3000);
        }
    }


    async function deleteVital(uid: string) {
        let response = await fetch(`${openEHRRestEndpoint}/ehr/${ehrId}/composition/${uid}`, {
            method: "DELETE",
            headers: {
                "Content-Type": "application/json"
            }
        });

        if (response.ok) {
            vitals = vitals.filter(v => v.uid !== uid);
            toastMessage = "Vitals deleted successfully!";
            showToast = true;
            clearTimeout(toastTimeout);
            toastTimeout = setTimeout(() => showToast = false, 3000);
        } else {
            console.error("Delete failed:", await response.text());
        }
    }

    loadVitals();
</script>


<!-- Toast Notification -->
{#if showToast}
    <div class="toast">
        <p>{toastMessage}</p>
    </div>
{/if}

<div class="flex w-full min-h-screen">
    <!-- Left: Form Section -->
    <div class="w-1/2 p-4 bg-white flex flex-col">
        <h2 class="text-2xl font-semibold mb-4">{updatingVitals ? 'Updating Nursing Form' : 'Nursing Form'}</h2>
        <mb-auto-form bind:this={form} webTemplate={webtemplate} class="flex-1"></mb-auto-form>
        <div class="mt-4 text-center">
            {#if updatingVitals}
                <button on:click={() => { updatingVitals = null; form.import({}); }}
                        class="mr-2 bg-gray-400 text-white text-lg font-semibold py-3 px-8 rounded-full shadow-md hover:bg-gray-500 transition duration-300">
                    Cancel
                </button>
            {/if}
            <button on:click={submitVitals}
                    class="bg-blue-500 text-white text-lg font-semibold py-3 px-8 rounded-full shadow-md hover:bg-blue-600 transition duration-300">
                Submit
            </button>
        </div>

    </div>


    <!-- Right: Previous Entries Section -->
    <div class="w-1/2 p-4 bg-gray-100 flex flex-col">
        <h2 class="text-2xl font-semibold mb-4">Previous Entries</h2>
        {#if vitals.length > 0}
            <div class="space-y-2 overflow-y-auto max-h-[80vh]">
                {#each vitals as vital (vital.uid)}
                    <div class="p-3 bg-white rounded shadow-sm relative group">
                        <!-- Top-right Edit/Delete buttons -->
                        <div class="absolute top-2 right-2 opacity-0 group-hover:opacity-100 transition-opacity duration-300">
                            <span on:click={() => editVital(vital.uid)}
                                  class="text-blue-500 cursor-pointer hover:text-blue-700 mr-4">
                                Edit
                            </span>
                            <span on:click={() => deleteVital(vital.uid)}
                                  class="text-red-500 cursor-pointer hover:text-red-700">
                                Delete
                            </span>
                        </div>

                        <div class="text-sm text-gray-600 mb-1">
                            {formatDateTime(vital.timestamp)}
                        </div>
                        <div class="flex text-sm font-mono">
                            <span class="w-16 font-bold">ID:</span>
                            <span class="truncate">{vital.uid}</span>
                        </div>
                    </div>
                {/each}
            </div>
        {:else}
            <p>No entries found.</p>
        {/if}
    </div>
</div>

<style>
    .toast {
        position: fixed;
        bottom: 20px;
        right: 20px;
        background-color: #444; /* neutral dark gray */
        color: white;
        padding: 10px 20px;
        border-radius: 5px;
        font-size: 16px;
        box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
        animation: slideIn 0.5s ease-out, fadeOut 3s 2.5s forwards;
        z-index: 50;
    }

    @keyframes slideIn {
        from {
            transform: translateY(50px);
            opacity: 0;
        }
        to {
            transform: translateY(0);
            opacity: 1;
        }
    }

    @keyframes fadeOut {
        from {
            opacity: 1;
        }
        to {
            opacity: 0;
        }
    }
</style>

