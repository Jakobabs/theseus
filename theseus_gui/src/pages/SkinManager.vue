<template>
  <Card class="header">
    <div class="iconified-input">
      <SearchIcon />
      <input v-model="search" type="text" placeholder="Search" class="search-input" />
      <Button @click="() => (search = '')">
        <XIcon />
      </Button>
    </div>
    <div class="labeled_button">
      <span>Sort by</span>
      <DropdownSelect
        v-model="sortBy"
        class="sort-dropdown"
        name="Sort Dropdown"
        :options="['Name', 'Date created', 'Date modified']"
        placeholder="Select..."
      />
    </div>
    <div class="labeled_button">
      <span>Filter by</span>
      <DropdownSelect
        v-model="filters"
        class="filter-dropdown"
        name="Filter Dropdown"
        :options="['Current user', 'All users']"
        placeholder="Select..."
      />
    </div>
  </Card>
  <div class="content">
    <Card>
      <p>Current Skin</p>
      <canvas id="skin_container" />
    </Card>
    <div class="row" >
      <Card class="instance-card-item button-base" >
        <Button @click="handleModal()">
          <PlusIcon />
        </Button>
        <div class="project-info">
          <p class="title">Add new skin</p>
        </div>
      </Card>
      <SkinSave
          v-for="skin in filteredResults" ref="skinComponents"
          :key="skin"
          :data="skin"
          @contextmenu.prevent.stop="(event) => handleRightClick(event, skin)"
          @set-skin="(data) => handleSkin(data.skin, data.cape, data.arms, 'upload')"
      />
      <ContextMenu ref="skinOptions" @option-clicked="handleOptionsClick">
        <template #use> <PlayIcon /> Use </template>
        <template #edit> <EyeIcon /> Edit </template>
        <template #duplicate> <ClipboardCopyIcon /> Duplicate </template>
        <template #delete> <TrashIcon /> Delete </template>
      </ContextMenu>
    </div>
  </div>

  <Modal ref="skinModal" class="modal" header="Change skin" :noblur="!themeStore.advancedRendering">
    <div v-if="!editSkin" class="modal-header">
      <Chips v-model="changeSkinType" :items="['from file', 'import from launcher']" />
    </div>
    <hr v-if="!editSkin" class="card-divider" />
    <div v-if="changeSkinType == 'from file'" class="modal-column">
      <div class="modal-row">
        <div class="modal-colum">
          <div class="image-upload">
            <Avatar :src="displaySkin" size="lg" />
          </div>
          <div class="image-input">
            <Button @click="openskin()">
              <FolderOpenIcon />
              Select skin
            </Button>
          </div>
          <div class="input-row">
            <p class="input-label">Name</p>
            <input
              v-model="skinName"
              autocomplete="off"
              class="text-input"
              type="text"
              maxlength="30"
              size="15"
            />
          </div>
          <div class="input-row">
            <p class="input-label">Arm style</p>
            <Chips v-model="skinArms" :items="['classic', 'slim']" />
          </div>
        </div>
        <canvas id="new_render" />
      </div>
      <div class="input-row">
        <p class="input-label">Cape</p>
        <Chips v-model="skinCape" :items="skinData.unlocked_capes" />
      </div>
      <div v-if="!editSkin" class="input-group push-right">
        <Button @click="skinModal.hide()">
          <XIcon />
          Cancel
        </Button>
        <Button :disabled="!validSkin || uploadingSkin" @click="handleSkin(newSkin, newCape, newArms, 'upload')">
          <UploadIcon />
          {{ uploadingSkin ? 'Uploading...' : 'Use' }}
        </Button>
        <Button
          :disabled="!validSkin || uploadingSkin || skinName.trim() == ''"
          @click="handleSkin(newSkin, newCape, newArms, 'save')"
        >
          <SaveIcon />
          Save
        </Button>
        <Button
          color="primary"
          :disabled="!validSkin || uploadingSkin || skinName.trim() == ''"
          @click="handleSkin(newSkin, newCape, newArms, 'saveupload')"
        >
          <PlusIcon v-if="!uploadingSkin" />
          {{ uploadingSkin ? 'Uploading...' : 'Save & Use' }}
        </Button>
      </div>
      <div v-else class="input-group push-right" >
        <Button @click="skinModal.hide()">
          <XIcon />
          Cancel
        </Button>
        <Button
          color="primary"
          :disabled="!validSkin || skinName.trim() == ''"
          @click="edit_skin_end"
        >
          <SaveIcon />
          Update
        </Button>
      </div>
    </div>
    <div v-else class="modal-body" >
      <div class="path-selection">
        <h3>.minecraft path</h3>
        <div class="path-input">
          <div class="iconified-input">
            <FolderOpenIcon />
            <input
              v-model="mojang.path"
              type="text"
              placeholder="Path to launcher"
            />
            <Button @click="() => (mojang.path = '')">
              <XIcon />
            </Button>
          </div>
          <Button icon-only @click="selectLauncherPath">
            <FolderSearchIcon />
          </Button>
          <Button icon-only @click="reload">
            <UpdatedIcon />
          </Button>
        </div>
      </div>
      <div class="table">
        <div class="table-head table-row">
          <div class="toggle-all table-cell">
            <Checkbox
              class="select-checkbox"
              :model-value="mojang.skinNames.every((child) => child.selected)"
              @update:model-value="
                (newValue) => mojang.skinNames.forEach((child) => (child.selected = newValue))
              "
            />
          </div>
          <div class="name-cell table-cell">All skins</div>
        </div>
        <div
          v-if="mojang.skinNames.length > 0"
          class="table-content"
        >
          <div
            v-for="skin in mojang.skinNames"
            :key="skin"
            class="table-row"
          >
            <div class="checkbox-cell table-cell">
              <Checkbox v-model="skin.selected" class="select-checkbox" />
            </div>
            <div class="name-cell table-cell">
              {{ skin.name }}
            </div>
          </div>
        </div>
        <div v-else class="table-content empty">No skins found</div>
      </div>
      <div>
        <InfoIcon /> Does not get capes. You may edit them after import
      </div>
      <div class="button-row">
        <Button
          :disabled="
            loading ||
            !Array.from(mojang.skinNames)
              .flatMap((e) => e)
              .some((e) => e.selected)
          "
          color="primary"
          @click="next"
        >
          {{
            loading
              ? 'Importing...'
              : Array.from(mojang.skinNames)
                  .flatMap((e) => e)
                  .some((e) => e.selected)
              ? `Import ${
                  Array.from(mojang.skinNames)
                    .flatMap((e) => e)
                    .filter((e) => e.selected).length
                } skins`
              : 'Select skins to import'
          }}
        </Button>
        <ProgressBar
          v-if="loading"
          :progress="(importedSkins / (totalSkins + 0.0001)) * 100"
        />
      </div>
    </div>
  </Modal>
</template>

<script setup>
import {
  Avatar,
  Card,
  Button,
  Modal,
  Chips,
  Checkbox,
  DropdownSelect,
  PlusIcon,
  SaveIcon,
  SearchIcon,
  UploadIcon,
  UpdatedIcon,
  PlayIcon,
  FolderOpenIcon,
  FolderSearchIcon,
  ClipboardCopyIcon,
  XIcon,
  InfoIcon,
  EyeIcon,
  TrashIcon,
} from 'omorphia'
import { ref, onMounted, watch, computed } from 'vue'
import ProgressBar from '@/components/ui/ProgressBar.vue'
import { handleError, useTheming } from '@/store/state.js'
import { open } from '@tauri-apps/api/dialog'
import { tauri } from '@tauri-apps/api'
import { selectedAccount } from '@/helpers/auth'
import dayjs from 'dayjs'
import SkinSave from '@/components/ui/SkinSave.vue'
import ContextMenu from '@/components/ui/ContextMenu.vue'

import {
  check_image,
  get_user_skin_data,
  get_mojang_launcher_path,
  get_mojang_launcher_names,
  set_skin,
  save_skin,
  import_skin,
  delete_skin,
  get_skins,
  get_render,
  get_cape_data,
  get_heads,
  set_cape,
} from '@/helpers/skin_manager.js'
import { IdleAnimation, SkinViewer, WalkingAnimation } from 'skinview3d'

const themeStore = useTheming()

const skinId = ref('')
const skinModal = ref(null)
const newSkin = ref(null)
const displaySkin = ref(null)
const validSkin = ref(false)
const uploadingSkin = ref(false)
const changeSkinType = ref('from file')
const skinName = ref('')
const skinData = ref(null)
const skinArms = ref('slim')
const skinCape = ref('no cape')
const currentRender = ref(null)
const newRender = ref(null)
const skinOptions = ref(null)
const skinComponents = ref([])
const editSkin = ref(false)
const loading = ref(false)
const importedSkins = ref(0)
const totalSkins = ref(0)
const mojang = ref({})
mojang.value.path = await get_mojang_launcher_path().catch(handleError)
mojang.value.skinNames = await get_mojang_launcher_names(mojang.value.path).catch(handleError)
console.log(mojang.value.skinNames)


const skinSaves = ref(await get_skins().catch(handleError))

const search = ref('')
const filters = ref('Current user')
const sortBy = ref('Name')

const filteredResults = computed(() => {
  let saves = skinSaves.value.filter((save) => {
    return save.name.toLowerCase().includes(search.value.toLowerCase())
  })

  if (filters.value === 'Current user') {
    saves = saves.filter((save) => {
      return save.user === selectedAccount.value.id
    })
  }

  if (sortBy.value === 'Name') {
    saves.sort((a, b) => {
      return a.name.localeCompare(b.name)
    })
  }

  if (sortBy.value === 'Date created') {
    saves.sort((a, b) => {
      return dayjs(b.created).diff(dayjs(a.created))
    })
  }

  if (sortBy.value === 'Date modified') {
    saves.sort((a, b) => {
      return dayjs(b.updated).diff(dayjs(a.updated))
    })
  }
  return saves
})

const handleRightClick = (event, item) => {
  const baseOptions = [
    {
      name: 'use',
      color: 'primary'
    },
    { type: 'divider' },
    { name: 'edit' },
    { name: 'duplicate' },
    { type: 'divider' },
    {
      name: 'delete',
      color: 'danger',
    },
  ]

  skinOptions.value.showMenu(
    event,
    item,
    baseOptions,
  )
}

const handleOptionsClick = async (args) => {
  switch (args.option) {
    case 'use':
      await handleSkin(args.item.skin, args.item.cape, args.item.arms, 'upload')
      break
    case 'edit':
      await edit_skin(args.item)
      break
    case 'duplicate':
      await duplicate_skin(args.item)
      break
    case 'delete':
      await delete_skin(args.item.id).catch(handleError)
      skinSaves.value = await get_skins().catch(handleError)
      break
  }
}

const selectLauncherPath = async () => {
  mojang.value.path = await open({ multiple: false, directory: true })

  if (mojang.value.path) {
    await reload()
  }
}

const reload = async () => {
  mojang.value.skinNames = get_mojang_launcher_names(mojang.value.path).catch(handleError)
}

const next = async () => {
  importedSkins.value = 0
  totalSkins.value = Array.from(mojang.value.skinNames)
    .flatMap((e) => e).filter((e) => e.selected).length
  loading.value = true
  for (const skin of mojang.value.skinNames.filter((skin) => skin.selected)) {
    await import_skin(skin.name, mojang.value.path, selectedAccount.value.id)
      .catch(handleError)
      .then(() => console.log(`Successfully Imported ${skin.name} from mojang`))
    skin.selected = false
    importedSkins.value++
  }
  skinSaves.value = await get_skins().catch(handleError)
  loading.value = false
  skinModal.value.hide()
  changeSkinType.value = 'from file'
}

const handleModal = async () => {
  skinClear()
  skinArms.value = 'classic'
  skinCape.value = 'no cape'
  skinName.value = ''
  skinId.value = ''
  editSkin.value = false
  skinModal.value.show()
}
  
const skinClear = async () => {
  validSkin.value = false
  displaySkin.value = null
  newSkin.value = null
  if (newRender.value) {
    newRender.value.resetSkin()
    newRender.value.resetCape()
  }
}

const handleSkin = async (skin, cape, arms, state) => {
  skinData.value.skin = skin
  skinData.value.cape = cape
  skinData.value.arms = arms

  if (state.includes('save')) await saveskin()
  if (state.includes('upload')) await uploadskin()

  skinModal.value.hide()
}

const edit_skin = async (data) => {
  editSkin.value = true
  displaySkin.value = data.skin
  skinName.value = data.name
  skinId.value = data.id
  validSkin.value = true
  await skinModal.value.show()
  newRender.value = new SkinViewer({
    canvas: document.getElementById('new_render'),
    width: 247.5,
    height: 330,
    preserveDrawingBuffer: true,
  })
  skinArms.value = data.arms
  skinCape.value = data.cape
  newRender.value.animation = new IdleAnimation()
  newRender.value.controls.enableZoom = false
  newRender.value.loadSkin(displaySkin.value, { model: await convert_arms(skinArms.value) })
}

const edit_skin_end = async () => {
  await handleSkin(displaySkin.value, skinCape.value, skinArms.value, 'save')
  skinClear()
  skinId.value = ''
  editSkin.value = false
}

const duplicate_skin = async (data) => {
  skinData.value.skin = data.skin
  skinData.value.cape = data.cape
  skinData.value.arms = data.arms
  skinId.value = ''

  await save_skin(data.user, skinData.value, data.name, data.model, skinId.value).catch(handleError)
  skinSaves.value = await get_skins().catch(handleError)
}

const openskin = async () => {
  newSkin.value = await open({
    multiple: false,
    filters: [
      {
        name: 'Image',
        extensions: ['png'],
      },
    ],
  })
  if (!newSkin.value) {
    skinClear()
    return
  }
  validSkin.value = await check_image(newSkin.value).catch(handleError)
  if (!validSkin.value) {
    skinClear()
    return
  }
  displaySkin.value = tauri.convertFileSrc(newSkin.value)
  if (!newRender.value) newRender.value = new SkinViewer({
    canvas: document.getElementById('new_render'),
    width: 247.5,
    height: 330,
    preserveDrawingBuffer: true,
  })
  newRender.value.animation = new IdleAnimation()
  newRender.value.controls.enableZoom = false
  newRender.value.loadSkin(displaySkin.value, { model: await convert_arms(skinArms.value) })
  if (skinCape.value !== 'no cape')
    newRender.value.loadCape(await get_cape_data(skinCape.value, 'url').catch(handleError))
}

const uploadskin = async () => {
  uploadingSkin.value = true
  let capeid = skinData.value.cape
  if (skinData.value.cape !== 'no cape') capeid = await get_cape_data(skinData.value.cape, 'id').catch(handleError)
  const uploadedCape = await set_cape(capeid, selectedAccount.value.access_token).catch(handleError)
  const uploadedSkin = await set_skin(skinData.value.skin, skinData.value.arms, selectedAccount.value).catch(handleError)

  if (uploadedSkin) {
    const arms = await convert_arms(skinData.value.arms)
    let skin = skinData.value.skin
    if (!skin.startsWith('data:image/png;base64,')) skin = tauri.convertFileSrc(skinData.value.skin)
    currentRender.value.loadSkin(skin, { model: arms })
  }
  if (uploadedCape) {
    if (skinData.value.cape == 'no cape') currentRender.value.resetCape()
    else {
      const capeurl = await get_cape_data(skinData.value.cape, 'url').catch(handleError)
      currentRender.value.loadCape(capeurl)
    }
  }
  uploadingSkin.value = false
  get_heads()
}

const saveskin = async () => {
  const model = await get_render(skinData.value).catch(handleError)
  await save_skin(selectedAccount.value.id, skinData.value, skinName.value.trim(), model, skinId.value).catch(handleError)
  skinSaves.value = await get_skins().catch(handleError)
}

const handleRender = async () => {
  skinData.value = await get_user_skin_data(selectedAccount.value.id).catch(handleError)
  const arms = await convert_arms(skinData.value.arms)
  const cape = await get_cape_data(skinData.value.cape, 'url').catch(handleError)
  currentRender.value = new SkinViewer({
    canvas: document.getElementById('skin_container'),
    width: 300,
    height: 400,
    skin: skinData.value.skin,
    model: arms,
    preserveDrawingBuffer: true,
  })
  if (cape !== 'no cape') currentRender.value.loadCape(cape)
  currentRender.value.animation = new WalkingAnimation()
  currentRender.value.animation.speed = 0.5
  currentRender.value.animation.headBobbing = false
  currentRender.value.controls.enableZoom = false
}

watch(selectedAccount, async (newAccount) => {
  skinData.value = await get_user_skin_data(newAccount.id).catch(handleError)
  currentRender.value.loadSkin(skinData.value.skin, {
    model: await convert_arms(skinData.value.arms),
  })
  if (skinData.value.cape == 'no cape') currentRender.value.resetCape()
  else currentRender.value.loadCape(await get_cape_data(skinData.value.cape, 'url').catch(handleError))
})

watch(skinArms, async (newArms) => {
  if (validSkin.value && newRender.value) {
    newRender.value.loadSkin(displaySkin.value, { model: await convert_arms(newArms) })
  }
})

watch(skinCape, async (newCape) => {
  if (validSkin.value && newRender.value) {
    if (newCape == 'no cape') newRender.value.resetCape()
    else newRender.value.loadCape(await get_cape_data(newCape, 'url').catch(handleError))
  }
})

watch(changeSkinType, async (type) => {
  if (type == 'import from launcher') {
    // get default launcher names
  }
})

async function convert_arms(arms) {
  if (arms == 'classic') arms = 'default'
  return arms
}

onMounted(() => {
  handleRender()
})
</script>

<style scoped lang="scss">
.skin-page {
  margin: 1rem;
}
.content {
  display: flex;
  flex-direction: row;
  width: 100%;
  padding: 1rem;
  gap: 1rem;

  -ms-overflow-style: none;
  scrollbar-width: none;

  &::-webkit-scrollbar {
    width: 0;
    background: transparent;
  }
}

.modal {
  position: absolute;
}

.image-upload {
  display: flex;
  gap: 1rem;
}

.image-input {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  justify-content: center;
}

.input-label {
  font-size: 1rem;
  font-weight: bolder;
  color: var(--color-contrast);
  margin-bottom: 0.5rem;
}

.modal-header {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  padding: var(--gap-lg);
  padding-bottom: 0;
}

.modal-column {
  display: flex;
  flex-direction: column;
  padding: var(--gap-lg);
  gap: var(--gap-sm);
}

.modal-row {
  display: flex;
  flex-direction: row;
  padding: var(--gap-sm);
  gap: var(--gap-sm);
}

.row {
  display: flex;
  flex-direction: row;
  align-items: flex-start;
  width: 100%;
  padding: 1rem;

  .divider {
    display: flex;
    justify-content: space-between;
    align-items: center;
    width: 100%;
    gap: 1rem;
    margin-bottom: 1rem;

    p {
      margin: 0;
      font-size: 1rem;
      white-space: nowrap;
      color: var(--color-contrast);
    }

    hr {
      background-color: var(--color-gray);
      height: 1px;
      width: 100%;
      border: none;
    }
  }
}

.header {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
  justify-content: space-between;
  gap: 1rem;
  align-items: inherit;
  margin: 1rem 1rem 0 !important;
  padding: 1rem;
  width: calc(100% - 2rem);

  .iconified-input {
    flex-grow: 1;

    input {
      min-width: 100%;
    }
  }

  .sort-dropdown {
    width: 10rem;
  }

  .filter-dropdown {
    width: 15rem;
  }

  .labeled_button {
    display: flex;
    flex-direction: row;
    align-items: center;
    gap: 0.5rem;
    white-space: nowrap;
  }
}

.instances {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(10rem, 1fr));
  width: 100%;
  gap: 1rem;
  margin-right: auto;
  scroll-behavior: smooth;
  overflow-y: auto;
}

.instance-card-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  padding: var(--gap-md);
  transition: 0.1s ease-in-out all !important; /* overrides Omorphia defaults */
  margin-bottom: 0;

  .mod-image {
    --size: 100%;

    width: 100% !important;
    height: auto !important;
    max-width: unset !important;
    max-height: unset !important;
    aspect-ratio: 1 / 1 !important;
  }

  .project-info {
    margin-top: 1rem;
    width: 100%;

    .title {
      color: var(--color-contrast);
      overflow: hidden;
      white-space: nowrap;
      text-overflow: ellipsis;
      width: 100%;
      margin: 0;
      font-weight: 600;
      font-size: 1rem;
      line-height: 110%;
      display: inline-block;
    }

    .description {
      color: var(--color-base);
      display: -webkit-box;
      -webkit-line-clamp: 2;
      -webkit-box-orient: vertical;
      overflow: hidden;
      font-weight: 500;
      font-size: 0.775rem;
      line-height: 125%;
      margin: 0.25rem 0 0;
      text-transform: capitalize;
      white-space: nowrap;
      text-overflow: ellipsis;
    }
  }
}

.modal-body {
  display: flex;
  flex-direction: column;
  padding: var(--gap-lg);
  gap: var(--gap-md);
}

.input-label {
  font-size: 1rem;
  font-weight: bolder;
  color: var(--color-contrast);
  margin-bottom: 0.5rem;
}

.text-input {
  width: 20rem;
}

.image-upload {
  display: flex;
  gap: 1rem;
}

.image-input {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  justify-content: center;
}

.warning {
  font-style: italic;
}

.versions {
  display: flex;
  flex-direction: row;
  gap: 1rem;
}

:deep(button.checkbox) {
  border: none;
}

.selector {
  max-width: 20rem;
}

.labeled-divider {
  text-align: center;
}

.labeled-divider:after {
  background-color: var(--color-raised-bg);
  content: 'Or';
  color: var(--color-base);
  padding: var(--gap-sm);
  position: relative;
  top: -0.5rem;
}

.info {
  display: flex;
  flex-direction: row;
  gap: 0.5rem;
  align-items: center;
}

.modal-header {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  padding: var(--gap-lg);
  padding-bottom: 0;
}

.path-selection {
  padding: var(--gap-xl);
  background-color: var(--color-bg);
  border-radius: var(--radius-lg);
  display: flex;
  flex-direction: column;
  gap: var(--gap-md);

  h3 {
    margin: 0;
  }

  .path-input {
    display: flex;
    align-items: center;
    width: 100%;
    flex-direction: row;
    gap: var(--gap-sm);

    .iconified-input {
      flex-grow: 1;
      :deep(input) {
        width: 100%;
        flex-basis: auto;
      }
    }
  }
}





.table {
  border: 1px solid var(--color-bg);
}

.table-row {
  grid-template-columns: min-content auto;
}

.table-content {
  max-height: calc(5 * (18px + 2rem));
  height: calc(5 * (18px + 2rem));
  overflow-y: auto;
}

.select-checkbox {
  button.checkbox {
    border: none;
  }
}

.button-row {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  gap: var(--gap-md);

  .transparent {
    padding: var(--gap-sm) 0;
  }
}

.empty {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.5rem;
  font-weight: bolder;
  color: var(--color-contrast);
}

.card-divider {
  margin: var(--gap-md) var(--gap-lg) 0 var(--gap-lg);
}
</style>