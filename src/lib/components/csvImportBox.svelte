<script lang="ts">
    import { onMount } from 'svelte';
    import { base } from '$app/paths';
    import { page } from '$app/state';
    import { sdk } from '$lib/stores/sdk';
    import { Dependencies } from '$lib/constants';
    import { goto, invalidate } from '$app/navigation';
    import { getProjectId } from '$lib/helpers/project';
    import { writable, type Writable } from 'svelte/store';
    import { addNotification } from '$lib/stores/notifications';
    import { Layout, Typography } from '@appwrite.io/pink-svelte';
    import { type Models, type Payload, Query } from '@appwrite.io/console';

    type ImportItem = {
        status: string;
        collection?: string;
    };

    type ImportItemsMap = Map<string, ImportItem>;

    /**
     * Keeps a track of the active and ongoing csv migrations.
     *
     * The structure is as follows -
     * `{ migrationId: { status: status, collection: collection } }`
     */
    const importItems: Writable<ImportItemsMap> = writable(new Map());

    async function showCompletionNotification(
        database: string,
        collection: string,
        payload: Payload
    ) {
        const isSuccess = payload.status === 'completed';
        const isError = !isSuccess && !!payload.errors;

        if (!isSuccess && !isError) return;

        let errorMessage = 'Import failed. Check your CSV for correct fields and required values.';
        if (isError && Array.isArray(payload.errors)) {
            try {
                // the `errors` is a list of json encoded string.
                errorMessage = JSON.parse(payload.errors[0]).message;
            } catch {
                // do nothing, fallback to default message.
            }
        }

        const type = isSuccess ? 'success' : 'error';
        const message = isError ? errorMessage : 'CSV import finished successfully.';
        const url = `${base}/project-${page.params.region}-${page.params.project}/databases/database-${database}/collection-${collection}`;

        addNotification({
            type,
            message,
            isHtml: true,
            buttons:
                isSuccess && collection !== page.params.collection
                    ? [{ name: 'View documents', method: () => goto(url) }]
                    : undefined
        });

        if (isSuccess) {
            await invalidate(Dependencies.DOCUMENTS);
        }
    }

    async function updateOrAddItem(importData: Payload | Models.Migration) {
        if (importData.source.toLowerCase() !== 'csv') return;

        const status = importData.status;
        const resourceId = importData.resourceId ?? '';
        const [databaseId, collectionId] = resourceId.split(':') ?? [];

        const current = $importItems.get(importData.$id);
        let collectionName = current?.collection ?? null;

        if (!collectionName && collectionId) {
            try {
                const collection = await sdk
                    .forProject(page.params.region, page.params.project)
                    .databases.getCollection(databaseId, collectionId);
                collectionName = collection.name;
            } catch {
                collectionName = null;
            }
        }

        importItems.update((items) => {
            const existing = items.get(importData.$id);

            const isDone = (s: string) => s === 'completed' || s === 'failed';
            const isInProgress = (s: string) => ['pending', 'processing', 'uploading'].includes(s);

            const shouldSkip =
                (existing && isDone(existing.status) && isInProgress(status)) ||
                existing?.status === status;

            if (shouldSkip) return items;

            const next = new Map(items);
            next.set(importData.$id, { status, collection: collectionName ?? undefined });
            return next;
        });

        if (status === 'completed' || status === 'failed') {
            await showCompletionNotification(databaseId, collectionId, importData);
        }
    }

    function clear() {
        importItems.update((items) => {
            items.clear();
            return items;
        });
    }

    function graphSize(status: string): number {
        switch (status) {
            case 'pending':
                return 10;
            case 'processing':
                return 30;
            case 'uploading':
                return 60;
            case 'completed':
            case 'failed':
                return 100;
            default:
                return 30;
        }
    }

    function text(status: string, collectionName = '') {
        const name = collectionName ? `<b>${collectionName}</b>` : '';
        switch (status) {
            case 'completed':
            case 'failed':
                return `Import to ${name} ${status}`;
            case 'processing':
                return `Importing CSV file${name ? ` to ${name}` : ''}`;
            default:
                return 'Preparing CSV for import...';
        }
    }

    onMount(() => {
        sdk.forProject(page.params.region, page.params.project)
            .migrations.list([
                Query.equal('source', 'CSV'),
                Query.equal('status', ['pending', 'processing'])
            ])
            .then((migrations) => {
                migrations.migrations.forEach(updateOrAddItem);
            });

        return sdk.forConsole.client.subscribe('console', (response) => {
            if (!response.channels.includes(`projects.${getProjectId()}`)) return;
            if (response.events.includes('migrations.*')) {
                updateOrAddItem(response.payload as Payload);
            }
        });
    });

    $: isOpen = true;
    $: showCsvImportBox = $importItems.size > 0;
</script>

{#if showCsvImportBox}
    <Layout.Stack direction="column" gap="l" alignItems="flex-end">
        <section class="upload-box">
            <header class="upload-box-header">
                <h4 class="upload-box-title">
                    <Typography.Text variant="m-500">
                        Importing documents ({$importItems.size})
                    </Typography.Text>
                </h4>
                <button
                    class="upload-box-button"
                    class:is-open={isOpen}
                    aria-label="toggle upload box"
                    on:click={() => (isOpen = !isOpen)}>
                    <span class="icon-cheveron-up" aria-hidden="true"></span>
                </button>
                <button
                    class="upload-box-button"
                    aria-label="close backup restore box"
                    on:click={clear}>
                    <span class="icon-x" aria-hidden="true"></span>
                </button>
            </header>

            {#each [...$importItems.entries()] as [key, value] (key)}
                <div class="upload-box-content" class:is-open={isOpen}>
                    <ul class="upload-box-list">
                        <li class="upload-box-item">
                            <section class="progress-bar u-width-full-line">
                                <div
                                    class="progress-bar-top-line u-flex u-gap-8 u-main-space-between">
                                    <Typography.Text>
                                        {@html text(value.status, value.collection)}
                                    </Typography.Text>
                                </div>
                                <div
                                    class="progress-bar-container"
                                    class:is-danger={value.status === 'failed'}
                                    style="--graph-size:{graphSize(value.status)}%">
                                </div>
                            </section>
                        </li>
                    </ul>
                </div>
            {/each}
        </section>
    </Layout.Stack>
{/if}

<style lang="scss">
    .upload-box-title {
        font-size: 11px;
    }

    .upload-box-content {
        min-width: 400px;
        max-width: 100vw;
    }

    .upload-box-button {
        display: flex;
        align-items: center;
        justify-content: center;
    }

    .progress-bar-container {
        height: 4px;

        &::before {
            height: 4px;
            background-color: var(--bgcolor-neutral-invert);
        }

        &.is-danger::before {
            height: 4px;
            background-color: var(--bgcolor-error);
        }
    }
</style>
