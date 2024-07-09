<script lang="ts">
    import { createEventDispatcher } from "svelte";
    import { waitForModalCompletion } from "../common/modal/modal-utils";
    import Modal from "../common/modal/Modal.svelte";
    import { asyncTxAction } from "./util/tx";
    import Form from "../common/Form.svelte";
    import MultiFactorAuthentication from "./MultiFactorAuthentication.svelte";
    import { messages } from "../global";
    import { getSpaceClient } from "./util/space-context";

    const dispatch = createEventDispatcher();
    const msg = messages.space.DeleteDirectory;
    const spaceClient = getSpaceClient();
    export let path: string;

    let mfaModal: Modal;
    let mfaModalOnClose;

    const submit = asyncTxAction(async (tx) => {
        // update to MFA session if necessary
        const dir = await tx.findDirectoryByPath(path);
        if (spaceClient.authorizer.requestDirectoryWriteAccess(dir) === "MFA_REQUIRED") {
            try {
                await waitForModalCompletion(mfaModal, (handler) => (mfaModalOnClose = handler));
            } catch (err) {
                dispatch("close");
                return;
            }
        }

        await tx.deleteDirectory(path);
        dispatch("close");
    });
</script>

<span>
    <h3>{msg.title}</h3>
    <Form {submit} cancel={() => dispatch('close')} submitTitle={msg.submitTitle}>
        <p>{msg.intro(path)}</p>
    </Form>

    <Modal bind:this={mfaModal}>
        <MultiFactorAuthentication on:close={mfaModalOnClose} />
    </Modal>
</span>

<style>
</style>
