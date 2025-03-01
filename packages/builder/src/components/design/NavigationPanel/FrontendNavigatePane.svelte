<script>
  import { onMount } from "svelte"
  import { goto, params } from "@roxi/routify"
  import {
    store,
    allScreens,
    selectedAccessRole,
    screenSearchString,
  } from "builderStore"
  import { roles } from "stores/backend"
  import ComponentNavigationTree from "components/design/NavigationPanel/ComponentNavigationTree/index.svelte"
  import Layout from "components/design/NavigationPanel/Layout.svelte"
  import NewLayoutModal from "components/design/NavigationPanel/NewLayoutModal.svelte"
  import {
    Icon,
    Modal,
    Select,
    Search,
    Tabs,
    Tab,
    notifications,
  } from "@budibase/bbui"

  export let showModal

  const tabs = [
    {
      title: "Screens",
      key: "screen",
    },
    {
      title: "Layouts",
      key: "layout",
    },
  ]
  let newLayoutModal
  $: selected = tabs.find(t => t.key === $params.assetType)?.title || "Screens"

  const navigate = ({ detail }) => {
    const { key } = tabs.find(t => t.title === detail)
    $goto(`../${key}`)
  }

  const updateAccessRole = event => {
    const role = event.detail

    // Select a valid screen with this new role - otherwise we'll not be
    // able to change role at all because ComponentNavigationTree will kick us
    // back the current role again because the same screen ID is still selected
    const firstValidScreenId = $allScreens.find(
      screen => screen.routing.roleId === role
    )?._id
    if (firstValidScreenId) {
      store.actions.screens.select(firstValidScreenId)
    }

    // Otherwise clear the selected screen ID so that the first new valid screen
    // can be selected by ComponentNavigationTree
    else {
      store.update(state => {
        state.selectedScreenId = null
        return state
      })
    }

    selectedAccessRole.set(role)
  }

  onMount(async () => {
    try {
      await store.actions.routing.fetch()
    } catch (error) {
      notifications.error("Error fetching routes")
    }
  })
</script>

<div class="title">
  <Tabs {selected} on:select={navigate}>
    <Tab title="Screens">
      <div class="tab-content-padding">
        <div class="role-select">
          <Select
            on:change={updateAccessRole}
            value={$selectedAccessRole}
            label="Filter by Access"
            getOptionLabel={role => role.name}
            getOptionValue={role => role._id}
            options={$roles}
          />
          <Search
            placeholder="Enter a route to search"
            label="Search Screens"
            bind:value={$screenSearchString}
          />
        </div>
        <div class="nav-items-container">
          <ComponentNavigationTree />
        </div>
      </div>
    </Tab>
    <Tab title="Layouts">
      <div class="tab-content-padding">
        {#each $store.layouts as layout, idx (layout._id)}
          <Layout {layout} border={idx > 0} />
        {/each}
        <Modal bind:this={newLayoutModal}>
          <NewLayoutModal />
        </Modal>
      </div>
    </Tab>
  </Tabs>
  <div class="add-button">
    <Icon
      hoverable
      name="AddCircle"
      on:click={selected === "Layouts" ? newLayoutModal.show() : showModal()}
    />
  </div>
</div>

<style>
  .title {
    display: flex;
    flex-direction: column;
    justify-content: flex-start;
    align-items: stretch;
    position: relative;
  }
  .add-button {
    position: absolute;
    top: var(--spacing-l);
    right: var(--spacing-xl);
  }

  .role-select {
    display: flex;
    flex-direction: column;
    justify-content: flex-start;
    align-items: stretch;
    margin-bottom: var(--spacing-m);
    gap: var(--spacing-m);
  }

  .tab-content-padding {
    padding: 0 var(--spacing-xl);
  }
</style>
