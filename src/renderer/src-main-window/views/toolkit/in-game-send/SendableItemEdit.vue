<template>
  <NCard size="small">
    <template #header>
      <span class="card-header-title">{{ t('title') }}</span>
    </template>
    <div class="template-edit">
      <div class="left-list">
        <NDropdown
          trigger="click"
          placement="bottom-start"
          :options="dropdownOptions"
          size="small"
          :theme-overrides="DROPDOWN_OVERRIDES"
          @select="handleDropdownSelect"
        >
          <NButton type="primary" secondary class="button-new" size="small">
            <template #icon>
              <NIcon>
                <AddIcon />
              </NIcon>
            </template>
            {{ t('newButton') }}
          </NButton>
        </NDropdown>
        <NInput
          v-if="igs2.settings.sendableItems.length > 0"
          v-model:value="filterText"
          :placeholder="t('filterPlaceholder')"
          class="filter-input"
          size="small"
          clearable
        >
          <template #prefix>
            <NIcon>
              <SearchIcon />
            </NIcon>
          </template>
        </NInput>
        <NVirtualList
          v-if="igs2.settings.sendableItems.length > 0"
          class="list"
          :padding-top="4"
          :item-size="30"
          key-field="id"
          :padding-bottom="4"
          :items="filteredItems"
        >
          <template #default="{ item }">
            <div
              @click="updateActiveItem(item.id)"
              class="list-item"
              :class="{ active: item.id === activeItemId }"
            >
              <NEllipsis class="name" :tooltip="{ placement: 'right' }">{{ item.name }}</NEllipsis>
              <div class="status-icons">
                <NPopover v-if="!item.isValid" placement="right">
                  <template #trigger>
                    <NIcon class="invalid-icon">
                      <Warning20FilledIcon />
                    </NIcon>
                  </template>
                  <div>
                    {{ t('errorTemplateInvalid') }}
                  </div>
                </NPopover>
                <NPopover v-else-if="executionErrors[item.id]" placement="right">
                  <template #trigger>
                    <NIcon class="invalid-icon">
                      <Warning20FilledIcon />
                    </NIcon>
                  </template>
                  <div :class="$style['error-message']">
                    <div :class="$style['error-title']">
                      {{ t('errorTemplateExecutionFailed') }}
                    </div>
                    <div :class="$style['error-divider']"></div>
                    <div :class="$style['error-content']">{{ executionErrors[item.id] }}</div>
                  </div>
                </NPopover>
                <NPopover v-else-if="item.enabled" placement="right">
                  <template #trigger>
                    <NIcon class="enabled-icon">
                      <CheckmarkIcon />
                    </NIcon>
                  </template>
                  <div>
                    {{ t('itemEnabled') }}
                  </div>
                </NPopover>
              </div>
            </div>
          </template>
        </NVirtualList>
        <div v-else class="empty">
          <div class="empty-text">
            {{ t('noSendableItem') }}
          </div>
        </div>
      </div>
      <div class="right-content">
        <template v-if="currentItem">
          <div class="header">
            <NInput
              size="small"
              @blur="handleSaveName"
              @keydown.enter="handleSaveName"
              v-if="isEditingName"
              v-model:value="tempName"
              ref="nameInputEl"
            />
            <div v-else class="title" @click="handleShowEditNameInput">
              <NEllipsis class="name">
                {{ currentItem.name }}
              </NEllipsis>
              <NIcon>
                <EditIcon />
              </NIcon>
            </div>
            <div class="actions">
              <NPopconfirm
                @positive-click="handleDelete"
                :positive-button-props="{
                  size: 'tiny',
                  type: 'error'
                }"
                :negative-button-props="{
                  size: 'tiny'
                }"
              >
                <template #trigger>
                  <NButton size="small" secondary type="error">
                    <template #icon>
                      <NIcon>
                        <DeleteIcon />
                      </NIcon>
                    </template>
                  </NButton>
                </template>
                <div style="max-width: 260px">{{ t('deletePopconfirm') }}</div>
              </NPopconfirm>
            </div>
          </div>
          <div class="control-items">
            <ControlItem :label="t('enabled.label')" :label-width="200">
              <NSwitch
                :value="currentItem.enabled"
                size="small"
                @update:value="
                  (value) => igs.updateSendableItem(currentItem!.id, { enabled: value })
                "
              />
            </ControlItem>
            <ControlItem
              :label="t('contentType.label')"
              :label-width="200"
              :label-description="t(`contentType.description.${currentItem.content.type}`)"
            >
              <NSelect
                style="width: 200px"
                :value="currentItem.content.type"
                :options="sendableItemTypeOptions"
                size="small"
                @update:value="handleSendableItemTypeChange"
              />
            </ControlItem>
            <ControlItem
              v-if="currentItem.content.type === 'template'"
              :label="t('template.label')"
              :label-width="200"
              :label-description="t('template.description')"
            >
              <NSelect
                size="small"
                style="width: 200px"
                :options="availableTemplates"
                :value="currentItem.content.templateId"
                @update:value="
                  (id) =>
                    igs.updateSendableItem(currentItem!.id, {
                      content: { type: 'template', templateId: id }
                    })
                "
              />
            </ControlItem>
            <ControlItem
              :label="t('sendShortcut.label')"
              :label-width="200"
              v-if="currentItem.content.type === 'plaintext'"
            >
              <ShortcutSelector
                :shortcut-id="currentItem.sendAllShortcut"
                :target-id="igs.getSendableItemShortcutTargetId(currentItem.id).all"
                @update:shortcut-id="
                  (id) => igs.updateSendableItem(currentItem!.id, { sendAllShortcut: id })
                "
              />
            </ControlItem>
            <ControlItem
              :label="t('sendAllShortcut.label')"
              :label-width="200"
              :label-description="t('sendAllShortcut.description')"
              v-if="currentItem.content.type === 'template'"
            >
              <ShortcutSelector
                :shortcut-id="currentItem.sendAllShortcut"
                :target-id="igs.getSendableItemShortcutTargetId(currentItem.id).all"
                @update:shortcut-id="
                  (id) => igs.updateSendableItem(currentItem!.id, { sendAllShortcut: id })
                "
              />
            </ControlItem>
            <ControlItem
              :label="t('sendAllyShortcut.label')"
              :label-width="200"
              :label-description="t('sendAllyShortcut.description')"
              v-if="currentItem.content.type === 'template'"
            >
              <ShortcutSelector
                :shortcut-id="currentItem.sendAllyShortcut"
                :target-id="igs.getSendableItemShortcutTargetId(currentItem.id).ally"
                @update:shortcut-id="
                  (id) => igs.updateSendableItem(currentItem!.id, { sendAllyShortcut: id })
                "
              />
            </ControlItem>
            <ControlItem
              :label="t('sendEnemyShortcut.label')"
              :label-width="200"
              :label-description="t('sendEnemyShortcut.description')"
              v-if="currentItem.content.type === 'template'"
            >
              <ShortcutSelector
                :shortcut-id="currentItem.sendEnemyShortcut"
                :target-id="igs.getSendableItemShortcutTargetId(currentItem.id).enemy"
                @update:shortcut-id="
                  (id) => igs.updateSendableItem(currentItem!.id, { sendEnemyShortcut: id })
                "
              />
            </ControlItem>
            <ControlItem
              :label="t('dryRun.label')"
              :label-width="200"
              :label-description="t('dryRun.description')"
              v-if="currentItem.content.type === 'template'"
            >
              <div class="button-group">
                <NButton
                  :disabled="!currentItem.content.templateId"
                  secondary
                  size="tiny"
                  @click="handleDryRun(currentItem.id, currentItem.content.templateId!, 'all')"
                >
                  {{ t('dryRun.all') }}
                </NButton>
                <NButton
                  :disabled="!currentItem.content.templateId"
                  secondary
                  size="tiny"
                  @click="handleDryRun(currentItem.id, currentItem.content.templateId!, 'ally')"
                >
                  {{ t('dryRun.ally') }}
                </NButton>
                <NButton
                  :disabled="!currentItem.content.templateId"
                  secondary
                  size="tiny"
                  @click="handleDryRun(currentItem.id, currentItem.content.templateId!, 'enemy')"
                >
                  {{ t('dryRun.enemy') }}
                </NButton>
              </div>
            </ControlItem>
          </div>
          <template v-if="currentItem.content.type === 'plaintext'">
            <div class="editor-header">
              <NPopover>
                <template #trigger>
                  <NButton size="small" secondary @click="handleRevert" :disabled="!changed">
                    <template #icon>
                      <NIcon>
                        <UndoIcon />
                      </NIcon>
                    </template>
                  </NButton>
                </template>
                <div>{{ t('revertButton') }}</div>
              </NPopover>
              <NPopover>
                <template #trigger>
                  <NButton
                    type="primary"
                    size="small"
                    secondary
                    @click="handleSave"
                    :disabled="!changed"
                  >
                    <template #icon>
                      <NIcon>
                        <SaveIcon />
                      </NIcon>
                    </template>
                  </NButton>
                </template>
                <div>{{ t('saveButton') }}</div>
              </NPopover>
            </div>
            <Codemirror
              class="editor"
              v-model="tempText"
              :style="{ flex: 1, height: 0, borderRadius: '2px', overflow: 'hidden' }"
              :autofocus="true"
              :indent-with-tab="true"
              :placeholder="t('plaintextPlaceholder')"
              :tab-size="2"
              :extensions="[vscodeDark]"
              @change="handleChange"
            />
          </template>
        </template>
        <template v-else>
          <div class="empty">
            <div class="empty-text">{{ t('noSendableItemSelected') }}</div>
          </div>
        </template>
      </div>
    </div>
  </NCard>
</template>

<script lang="ts" setup>
import ControlItem from '@renderer-shared/components/ControlItem.vue'
import { useInstance } from '@renderer-shared/shards'
import { InGameSendRenderer } from '@renderer-shared/shards/in-game-send'
import { useInGameSendStore } from '@renderer-shared/shards/in-game-send/store'
import { vscodeDark } from '@uiw/codemirror-theme-vscode'
import {
  Add as AddIcon,
  Checkmark as CheckmarkIcon,
  Delete as DeleteIcon,
  Edit as EditIcon,
  Save as SaveIcon,
  Search as SearchIcon,
  Undo as UndoIcon
} from '@vicons/carbon'
import { Warning20Filled as Warning20FilledIcon } from '@vicons/fluent'
import { useTranslation } from 'i18next-vue'
import {
  DialogReactive,
  NButton,
  NCard,
  NDropdown,
  NEllipsis,
  NIcon,
  NInput,
  NPopconfirm,
  NPopover,
  NScrollbar,
  NSelect,
  NSwitch,
  NVirtualList,
  useDialog,
  useMessage
} from 'naive-ui'
import { computed, h, nextTick, ref, shallowReactive, shallowRef, useTemplateRef, watch } from 'vue'
import { Codemirror } from 'vue-codemirror'

import ShortcutSelector from '@main-window/components/ShortcutSelector.vue'

import { DROPDOWN_OVERRIDES } from './style-overrides'

// 还是直接复制一份组件好用
const { t } = useTranslation('renderer', { keyPrefix: 'SendableItemEdit' })

const igs2 = useInGameSendStore()
const igs = useInstance(InGameSendRenderer)

const message = useMessage()
const activeItemId = ref<string | null>(null)

const dropdownOptions = computed(() => [
  {
    label: t('sendableItemPresets.plaintext'),
    key: 'plaintext'
  },
  {
    label: t('sendableItemPresets.template'),
    key: 'template'
  }
])

const sendableItemTypeOptions = computed(() => [
  {
    label: t('sendableItemPresets.plaintext'),
    value: 'plaintext'
  },
  {
    label: t('sendableItemPresets.template'),
    value: 'template'
  }
])

const availableTemplates = computed(() => {
  return igs2.settings.templates.map((t) => ({
    label: t.name,
    value: t.id
  }))
})

const handleSendableItemTypeChange = (value: string) => {
  if (currentItem.value) {
    if (value === 'plaintext') {
      igs.updateSendableItem(currentItem.value.id, { content: { type: 'plaintext', content: '' } })
    } else if (value === 'template') {
      igs.updateSendableItem(currentItem.value.id, {
        content: { type: 'template', templateId: null }
      })
    }
  }
}

const handleDropdownSelect = async (key: string) => {
  if (key === 'plaintext') {
    const newItem = await igs.createSendableItem({
      content: {
        type: 'plaintext',
        content: ''
      }
    })

    if (newItem) {
      updateActiveItem(newItem.id)
    }
  } else if (key === 'template') {
    const newItem = await igs.createSendableItem({
      content: {
        type: 'template',
        templateId: null
      }
    })

    if (newItem) {
      updateActiveItem(newItem.id)
    }
  }
}

const isEditingName = ref(false)
const tempName = ref('')
const tempText = ref('')

const sendableItems = computed(() => {
  const validMap = new Map<string, boolean>()
  for (const template of igs2.settings.templates) {
    validMap.set(template.id, template.isValid)
  }

  return igs2.settings.sendableItems.map((item) => {
    let isValid = true
    if (item.content.type === 'template') {
      if (item.content.templateId) {
        isValid = validMap.get(item.content.templateId) ?? false
      } else {
        isValid = false
      }
    }

    return { ...item, isValid }
  })
})

const currentItem = computed(() => {
  return sendableItems.value.find((item) => item.id === activeItemId.value)
})

const changed = ref(false)
const filterText = ref('')
const filteredItems = computed(() => {
  return sendableItems.value.filter(
    (item) => item.name.includes(filterText.value) || item.id.includes(filterText.value)
  )
})

const updateActiveItem = (id: string) => {
  activeItemId.value = id
}

watch(
  () => currentItem.value,
  (item) => {
    if (item) {
      tempText.value = item.content.type === 'plaintext' ? item.content.content : ''
      isEditingName.value = false
      changed.value = false
    }
  },
  { immediate: true }
)

const nameInputEl = useTemplateRef('nameInputEl')
const handleShowEditNameInput = () => {
  if (currentItem.value) {
    isEditingName.value = true
    tempName.value = currentItem.value.name
    nextTick(() => {
      nameInputEl.value?.focus()
    })
  }
}

const handleSaveName = async () => {
  if (currentItem.value) {
    igs.updateSendableItem(currentItem.value.id, { name: tempName.value })
    isEditingName.value = false
  }
}

const handleRevert = () => {
  if (currentItem.value && currentItem.value.content.type === 'plaintext') {
    tempText.value = currentItem.value.content.content
    changed.value = false
  }
}

const handleSave = () => {
  if (currentItem.value && currentItem.value.content.type === 'plaintext') {
    igs.updateSendableItem(currentItem.value.id, {
      content: { type: 'plaintext', content: tempText.value }
    })
    message.success(() => t('saveSuccess', { name: currentItem.value!.name }))
  }
}

const handleChange = (_: string, __: any) => {
  changed.value = true
}

const handleDelete = () => {
  if (currentItem.value) {
    let name = currentItem.value.name
    igs.removeSendableItem(currentItem.value.id)
    message.success(() => t('deleteSuccess', { name }))
  }
}

watch(
  () => igs2.settings.sendableItems,
  (sendableItems) => {
    nextTick(() => {
      if (!currentItem.value && sendableItems.length > 0) {
        updateActiveItem(sendableItems[0].id)
      }
    })
  },
  { immediate: true }
)

const dialog = useDialog()
const dialogRef = shallowRef<DialogReactive>()

const executionErrors = shallowReactive<Record<string, string>>({})

igs.onTemplateExecutionFailed(({ templateId, error }) => {
  executionErrors[templateId] = error

  // 不会堆积太多
  for (const id of Object.keys(executionErrors)) {
    if (!sendableItems.value.find((item) => item.id === id)) {
      delete executionErrors[id]
    }
  }
})

igs.onTemplateExecutionSucceeded(({ templateId }) => {
  delete executionErrors[templateId]
})

const handleDryRun = async (id: string, templateId: string, target: 'ally' | 'enemy' | 'all') => {
  const result = await igs.getDryRunResult(templateId, target)

  if (result.error) {
    message.error(() => t('dryRunError'))
    executionErrors[id] = result.error

    return
  } else {
    delete executionErrors[id]
  }

  dialogRef.value?.destroy()
  dialogRef.value = dialog.create({
    type: 'info',
    title: 'Dry Run',
    content: () =>
      h(
        NScrollbar,
        {
          style: { maxHeight: '80vh' }
        },
        () =>
          result.messages.length > 0
            ? h(
                'div',
                {
                  style: { userSelect: 'text' }
                },
                result.messages.map((line) => h('div', line))
              )
            : h(
                'div',
                {
                  style: {
                    color: '#fff8'
                  }
                },
                '(' + t('dryRunEmpty') + ')'
              )
      )
  })
}
</script>

<style lang="less" scoped>
.template-edit {
  display: flex;
  height: 600px;
  border: 1px solid #fff1;
}

.left-list {
  display: flex;
  flex-direction: column;
  width: 200px;
  height: 100%;
  border-right: 1px solid #fff1;
  flex-shrink: 0;

  .button-new {
    margin-bottom: 8px;
    align-self: flex-start;
  }

  .filter-input {
    margin-bottom: 8px;
  }

  .list {
    flex-grow: 1;
    border: 1px solid #fff1;
    border-radius: 2px;

    .list-item {
      display: flex;
      align-items: center;
      border-radius: 2px;
      height: 28px;
      padding: 0 8px;
      box-sizing: border-box;
      cursor: pointer;
      transition: background-color 0.2s;
      margin: 0 4px 2px 4px;
      font-size: 12px;

      &:hover {
        background-color: #fff1;
      }

      &.active {
        background-color: #fff2;
      }

      .status-icons {
        margin-left: auto;
      }

      .enabled-icon {
        font-size: 14px;
        color: #00ff00;
      }

      .invalid-icon {
        font-size: 14px;
        color: #ffd900;
      }
    }
  }

  .empty {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;

    .empty-text {
      font-size: 16px;
      color: #fff1;
    }
  }
}

.right-content {
  flex: 1;
  width: 0;
  height: 100%;
  display: flex;
  flex-direction: column;

  .header {
    display: flex;
    gap: 8px;
    align-items: center;

    .title {
      font-size: 16px;
      font-weight: bold;
      flex-grow: 1;
      height: 28px;
      width: 0;

      display: flex;
      align-items: center;
      gap: 4px;
      cursor: pointer;
      transition: color 0.2s;

      .name {
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
      }

      &:hover {
        color: #fff;
      }
    }

    .actions {
      display: flex;
      gap: 8px;
      align-items: center;
    }

    margin-bottom: 16px; // here 16px
  }

  .control-items {
    display: flex;
    flex-direction: column;
    gap: 12px;
    margin-bottom: 16px;
  }

  .button-group {
    display: flex;
    gap: 4px;
    align-items: center;
  }

  .editor-header {
    display: flex;
    gap: 8px;
    align-items: center;
    justify-content: flex-end;
    margin-bottom: 4px;
  }

  .editor {
    flex: 1;
    border: 1px solid #fff1;
    border-radius: 2px;
  }

  .empty {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;

    .empty-text {
      font-size: 16px;
      color: #fff1;
    }
  }
}

.left-list,
.right-content {
  padding: 8px;
  box-sizing: border-box;
}

.one-time-player-filter {
  margin-bottom: 16px;

  .checkbox {
    display: flex;
    align-items: center;
    gap: 8px;
  }
}
</style>

<style lang="less" module>
.error-message {
  .error-title {
    font-size: 12px;
  }

  .error-divider {
    height: 1px;
    background-color: #fff2;
    margin: 8px 0;
  }

  .error-content {
    font-size: 12px;
    white-space: pre-wrap;
  }
}
</style>
