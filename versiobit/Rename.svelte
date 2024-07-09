<script lang="ts">
    import { createEventDispatcher } from "svelte";
    import { asyncTxAction } from "./util/tx";
    import Form from "../common/Form.svelte";
    import { UserError } from "../common/UserError";
    import { messages } from "../global";

    const dispatch = createEventDispatcher();
    const msg = messages.space.Rename;
    export let path: string;
    export let type: string;

    let name = path.substring(path.lastIndexOf("/") + 1);

    const pattern = '^((?![<>:"/\\|?*]).)+$';

    const submit = asyncTxAction(async (tx) => {
        const destPath = path.substring(0, path.lastIndexOf("/") + 1) + name.normalize();

        if (type === "file") {
            if (await tx.findFileByPathOrNull(destPath)) {
                throw new UserError(msg.errorFileAlreadyExists);
            }
            await tx.moveFile(path, destPath);
        }
        if (type === "directory") {
            if (await tx.findDirectoryByPathOrNull(destPath)) {
                throw new UserError(msg.errorFolderAlreadyExists);
            }
            await tx.renameDirectory(path, name.normalize());
        }
        dispatch("close");
    });
</script>

<span>
    <h3>{msg.title}</h3>

    <Form {submit} cancel={() => dispatch('close')}>
        <label for="name">{msg.nameLabel}</label><br />
        <input name="name" type="text" bind:value={name} required {pattern} maxlength="200" />
    </Form>
</span>

<style>
    input {
        width: 100%;
        max-width: 40rem;
    }
</style>
