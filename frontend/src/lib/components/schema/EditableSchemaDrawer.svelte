<script lang="ts">
	import type { Schema } from '$lib/common'
	import { GripVertical, Pen } from 'lucide-svelte'
	import EditableSchemaForm from '../EditableSchemaForm.svelte'
	import { Drawer, DrawerContent } from '../common'
	import AddProperty from './AddProperty.svelte'
	import { dragHandle, dragHandleZone } from '@windmill-labs/svelte-dnd-action'
	import { flip } from 'svelte/animate'
	import { generateRandomString } from '$lib/utils'
	import Button from '$lib/components/common/button/Button.svelte'
	import { createEventDispatcher, tick } from 'svelte'
	import Label from '../Label.svelte'

	export let schema: Schema | undefined | any
	export let parentId: string | undefined = undefined

	const flipDurationMs = 200

	const dispatch = createEventDispatcher()

	let addProperty: AddProperty | undefined = undefined
	let schemaFormDrawer: Drawer | undefined = undefined
	let editableSchemaForm: EditableSchemaForm | undefined = undefined

	$: items = ((schema?.order ?? Object.keys(schema.properties ?? {}))?.map((item, index) => {
		return { value: item, id: generateRandomString() }
	}) ?? []) as Array<{
		value: string
		id: string
	}>

	function handleConsider(e) {
		const { items: newItems } = e.detail

		items = newItems
	}

	function handleFinalize(e) {
		const { items: newItems } = e.detail

		items = newItems

		const keys = items.map((item) => item.value)

		schema.properties = keys.reduce((acc, key) => {
			acc[key] = schema.properties[key]
			return acc
		}, {})

		schema.order = keys

		schema = { ...schema }
		dispatch('change', schema)
	}

	const yOffset = 49
</script>

<AddProperty on:change bind:schema bind:this={addProperty} />

<div
	use:dragHandleZone={{
		items,
		flipDurationMs,
		dropTargetStyle: {},
		type: parentId ? `app-editor-fields-${parentId}` : 'app-editor-fields'
	}}
	on:consider={handleConsider}
	on:finalize={handleFinalize}
	class="gap-1 flex flex-col mt-2"
>
	{#if items?.length > 0}
		{#each items as item (item.id)}
			<!-- svelte-ignore a11y-no-noninteractive-tabindex -->
			<div
				animate:flip={{ duration: 200 }}
				class="w-full flex flex-col justify-between border items-center py-1 px-2 rounded-md bg-surface text-sm"
			>
				{#if schema.properties?.[item.value]}
					<div class="flex flex-row justify-between items-center w-full">
						{`${item.value}${
							schema.properties?.[item.value]?.title
								? ` (title: ${schema.properties?.[item.value]?.title})`
								: ''
						} `}
						<!-- svelte-ignore a11y-no-noninteractive-tabindex -->
						<!-- svelte-ignore a11y-no-static-element-interactions -->
						<div class="flex flex-row gap-1 item-center h-full justify-center">
							<Button
								iconOnly
								size="xs2"
								color="light"
								startIcon={{ icon: Pen }}
								on:click={() => {
									schemaFormDrawer?.openDrawer()

									tick().then(() => {
										editableSchemaForm?.openField(item.value)
									})
								}}
							/>

							<div class="cursor-move flex items-center handle" use:dragHandle>
								<GripVertical size={16} />
							</div>
						</div>
					</div>

					{#if schema.properties[item.value]?.type === 'object'}
						<div class="flex flex-col w-full mt-2">
							<Label label="Nested Properties">
								<svelte:self
									on:change={() => (schema = schema)}
									schema={schema.properties[item.value]}
									parentId={item.value}
								/>
							</Label>
						</div>
					{/if}
				{:else}
					<div class="text-tertiary"> Value is undefined </div>
				{/if}
			</div>
		{/each}
	{/if}
</div>

<Drawer bind:this={schemaFormDrawer} size="1200px">
	<DrawerContent noPadding title="UI Customisation" on:close={schemaFormDrawer.closeDrawer}>
		<svelte:fragment slot="actions">
			<AddProperty bind:schema />
		</svelte:fragment>
		<EditableSchemaForm
			bind:this={editableSchemaForm}
			bind:schema
			isAppInput
			on:edit={(e) => {
				addProperty?.openDrawer(e.detail)
			}}
			on:delete={(e) => {
				addProperty?.handleDeleteArgument([e.detail])
			}}
			offset={yOffset}
			lightweightMode
			dndType="drawer"
		/>
	</DrawerContent>
</Drawer>
