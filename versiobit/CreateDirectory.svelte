<script lang="ts">
    import { createEventDispatcher } from "svelte";
    import { waitForModalCompletion } from "../common/modal/modal-utils";
    import Modal from "../common/modal/Modal.svelte";
    import { asyncTxAction } from "./util/tx";
    import Form from "../common/Form.svelte";
    import { UserError } from "../common/UserError";
    import MultiFactorAuthentication from "./MultiFactorAuthentication.svelte";
    import { messages } from "../global";
    import { getSpaceClient } from "./util/space-context";

    const dispatch = createEventDispatcher();
    const msg = messages.space.CreateDirectory;
    const spaceClient = getSpaceClient();
    export let currentPath: string;
    export let owner = 10;

    // bug: Input is marked as invalid for inital null value
    // https://github.com/sveltejs/svelte/issues/5814

    let name = null;
    let mfaModal: Modal;
    let mfaModalOnClose;

    const submit = asyncTxAction(async (tx) => {
        const path = `${currentPath ? `${currentPath}/` : ""}${name.normalize()}`;

        if (await tx.findDirectoryByPathOrNull(path)) {
            throw new UserError(msg.errorFolderAlreadyExists);
        }

        // update to MFA session if necessary
        if (spaceClient.authorizer.requestDirectoryWriteAccessForOwner(owner) === "MFA_REQUIRED") {
            try {
                await waitForModalCompletion(mfaModal, (handler) => (mfaModalOnClose = handler));
            } catch (err) {
                dispatch("close");
                return;
            }
        }

        try {
            await tx.putDirectory(path, owner);
            dispatch("close");
        } catch (error) {
            const errorMessage = msg.backendError[error._errorCode];
            if (errorMessage) {
                throw new UserError(errorMessage);
            } else {
                throw error;
            }
        }
    });
</script>

<span>
    <h3>{msg.title}</h3>

    <Form {submit} cancel={() => dispatch('close')}>
        <div class="dl-form-row">
            <label>
                {msg.nameLabel}<br />
                <input type="text" bind:value={name} required pattern='^((?![<>:"\/\\|?*]).)+$' maxlength="200" />
            </label>
        </div>

        <div class="dl-form-row">
            {#if currentPath}
                <p>{msg.folderLabel(currentPath)}</p>
            {/if}
        </div>

        {#if owner != null}
            <div class="dl-form-row">
                <label for="owner">{msg.ownerLabel}</label>
                <div>
                    <select name="owner" bind:value={owner}>
                        <option value={10}>{msg.owner10.name}</option>
                        <option value={20}>{msg.owner20.name}</option>
                        <option value={30}>{msg.owner30.name}</option>
                        <option value={40}>{msg.owner40.name}</option>
                    </select>
                    <p>{msg[`owner${owner}`].description}</p>
                </div>
            </div>
        {/if}
    </Form>

    <Modal bind:this={mfaModal}>
        <MultiFactorAuthentication on:close={mfaModalOnClose} />
    </Modal>
</span>

<style>
    .dl-form-row {
        margin-top: 1rem;
        margin-bottom: 1.5rem;
    }
</style>
