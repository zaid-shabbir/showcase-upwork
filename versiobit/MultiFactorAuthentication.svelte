<script lang="ts">
    import { createEventDispatcher } from "svelte";
    import Form from "../common/Form.svelte";
    import { UserError } from "../common/UserError";
    import { appContext, messages } from "../global";
    import { updateTx } from "./util/tx";

    const dispatch = createEventDispatcher();
    const msg = messages.space.MultiFactorAuthentication;

    let step = "requestCode";
    let code = "";

    async function requestCode() {
        const response = await fetch("/request-mfa-code.json", {
            method: "POST",
            credentials: "include",
            headers: { "content-type": "application/json" },
            body: "",
        });

        if (response.status === 200) {
            step = "confirmCode";
        } else if (response.status === 400) {
            const responseBody = await response.json();
            const errorMessage = msg.error[responseBody.errorCode];
            if (errorMessage) {
                throw new UserError(errorMessage);
            } else {
                throw responseBody;
            }
        } else {
            throw new Error(`Requesting MFA code failed with status ${response.status}`);
        }
    }

    async function confirmCode() {
        const errorCode = await appContext.getSessionManager().performMfaUpgrade(code);
        if (!errorCode) {
            await updateTx();
            dispatch("close", { success: true });
        } else {
            const errorMessage = msg.error[errorCode];
            throw new UserError(errorMessage);
        }
    }
</script>

<span>
    <h3>{msg.title}</h3>
    {#if step === 'requestCode'}
        <Form submit={requestCode} cancel={() => dispatch('close')} submitTitle={msg.requestCode.submitTitle}>
            <p>{msg.requestCode.intro}</p>
        </Form>
    {:else}
        <Form submit={confirmCode} cancel={() => dispatch('close')} submitTitle={msg.confirmCode.submitTitle}>
            <p>{msg.confirmCode.intro}</p>
            <label>{msg.confirmCode.codeLabel}<br /><input type="text" bind:value={code} required /></label>
        </Form>
    {/if}
</span>

<style>
</style>
