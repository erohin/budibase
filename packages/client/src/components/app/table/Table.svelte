<script>
  import { getContext } from "svelte"
  import { Table } from "@budibase/bbui"
  import SlotRenderer from "./SlotRenderer.svelte"
  import { UnsortableTypes } from "../../../constants"
  import { onDestroy } from "svelte"

  export let dataProvider
  export let columns
  export let showAutoColumns
  export let rowCount
  export let quiet
  export let size
  export let linkRows
  export let linkURL
  export let linkColumn
  export let linkPeek
  export let allowSelectRows
  export let compact

  const component = getContext("component")
  const { styleable, getAction, ActionTypes, routeStore, rowSelectionStore } =
    getContext("sdk")
  const customColumnKey = `custom-${Math.random()}`
  const customRenderers = [
    {
      column: customColumnKey,
      component: SlotRenderer,
    },
  ]
  let selectedRows = []
  $: hasChildren = $component.children
  $: loading = dataProvider?.loading ?? false
  $: data = dataProvider?.rows || []
  $: fullSchema = dataProvider?.schema ?? {}
  $: fields = getFields(fullSchema, columns, showAutoColumns)
  $: schema = getFilteredSchema(fullSchema, fields, hasChildren)
  $: setSorting = getAction(
    dataProvider?.id,
    ActionTypes.SetDataProviderSorting
  )
  $: {
    rowSelectionStore.actions.updateSelection(
      $component.id,
      selectedRows.map(row => row._id)
    )
  }

  const getFields = (schema, customColumns, showAutoColumns) => {
    // Check for an invalid column selection
    let invalid = false
    customColumns?.forEach(column => {
      const columnName = typeof column === "string" ? column : column.name
      if (schema[columnName] == null) {
        invalid = true
      }
    })

    // Use column selection if it exists
    if (!invalid && customColumns?.length) {
      return customColumns
    }

    // Otherwise generate columns
    let columns = []
    let autoColumns = []
    Object.entries(schema).forEach(([field, fieldSchema]) => {
      if (!fieldSchema?.autocolumn) {
        columns.push(field)
      } else if (showAutoColumns) {
        autoColumns.push(field)
      }
    })
    return columns.concat(autoColumns)
  }

  const getFilteredSchema = (schema, fields, hasChildren) => {
    let newSchema = {}
    if (hasChildren) {
      newSchema[customColumnKey] = {
        displayName: null,
        order: 0,
        sortable: false,
        divider: true,
        width: "auto",
      }
    }

    fields.forEach(field => {
      const columnName = typeof field === "string" ? field : field.name
      if (!schema[columnName]) {
        return
      }
      newSchema[columnName] = schema[columnName]
      if (UnsortableTypes.includes(schema[columnName].type)) {
        newSchema[columnName].sortable = false
      }

      // Add additional settings like width etc
      if (typeof field === "object") {
        newSchema[columnName] = {
          ...newSchema[columnName],
          ...field,
        }
      }
    })
    return newSchema
  }

  const onSort = e => {
    setSorting({
      column: e.detail.column,
      order: e.detail.order,
    })
  }

  const onClick = e => {
    if (!linkRows || !linkURL) {
      return
    }
    const col = linkColumn || "_id"
    const id = e.detail?.[col]
    if (!id) {
      return
    }
    const split = linkURL.split("/:")
    routeStore.actions.navigate(`${split[0]}/${id}`, linkPeek)
  }

  onDestroy(() => {
    rowSelectionStore.actions.updateSelection($component.id, [])
  })
</script>

<div use:styleable={$component.styles} class={size}>
  <Table
    {data}
    {schema}
    {loading}
    {rowCount}
    {quiet}
    {compact}
    {customRenderers}
    allowSelectRows={!!allowSelectRows}
    bind:selectedRows
    allowEditRows={false}
    allowEditColumns={false}
    showAutoColumns={true}
    disableSorting
    autoSortColumns={!columns?.length}
    on:sort={onSort}
    on:click={onClick}
  >
    <slot />
  </Table>
  {#if allowSelectRows && selectedRows.length}
    <div class="row-count">
      {selectedRows.length} row{selectedRows.length === 1 ? "" : "s"} selected
    </div>
  {/if}
</div>

<style>
  div {
    background-color: var(--spectrum-alias-background-color-secondary);
  }

  .row-count {
    margin-top: var(--spacing-l);
  }
</style>
