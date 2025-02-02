<template>
  <Splitpanes horizontal>
    <pane min-size="10" class="directory-div" :theme="theme">
      <a-row class="header-directory" align="center" justify="space-between">
        <a-space>
          <a-typography-text class="title-text-directory" strong>FILE EXPLORER</a-typography-text>
        </a-space>
        <a-space align="center" class="control-btns" v-if="!readOnly">
          <a-row>
            <a-tooltip title="New folder">
              <a-button
                v-if="!isMulti"
                type="text"
                class="icon-btn-dir"
                @click="() => onNewDirectory('folder')"
              >
                <img src="/icons/folder-add.svg" class="icon-select" />
              </a-button>
            </a-tooltip>
            <a-tooltip title="New file">
              <a-button
                v-if="!isMulti"
                type="text"
                class="icon-btn-dir"
                @click="() => onNewDirectory('file')"
              >
                <img src="/icons/file-add.svg" class="icon-select" />
              </a-button>
            </a-tooltip>
            <a-tooltip title="New image">
              <a-button
                v-if="!isMulti"
                type="text"
                class="icon-btn-dir"
                @click="() => (openImage = true)"
              >
                <img src="/icons/file-image.svg" class="icon-select" />
              </a-button>
            </a-tooltip>
          </a-row>
          <a-row>
            <a-tooltip title="Delete">
              <a-button type="text" class="icon-btn-dir" @click="() => onDelete(selectedKeys)">
                <img src="/icons/trash.svg" class="icon-select" />
              </a-button>
            </a-tooltip>
          </a-row>
        </a-space>
      </a-row>
      <div class="tree-container">
        <a-directory-tree
          mode="inline"
          :style="{ height: '100%', borderRight: 0 }"
          v-model:selectedKeys="selectedKeys"
          v-model:expandedKeys="expandedKeys"
          :tree-data="files.children"
          multiple
          @select="onChangeSelectedKeys"
        >
          <template #title="{ dataRef }">
            <a-row justify="space-between" class="row-directory" :class="{ save: dataRef.isSave }">
              <span
                v-if="dataRef.id !== 'input'"
                :style="{ overflow: 'hidden', maxWidth: 'calc(100% - 50px)' }"
                :title="dataRef.title"
                >{{ dataRef.title }}</span
              >
              <span v-if="dataRef.id === 'input'">
                <a-input
                  size="small"
                  v-model:value="name"
                  ref="input"
                  @blur="() => handleBlurName(dataRef)"
                  @pressEnter="() => input.blur()"
                ></a-input>
              </span>
              <a-space>
                <span style="font-weight: 600" v-if="!dataRef.isSave">U</span>
                <a-dropdown :trigger="['click']" v-if="dataRef.id !== 'input'">
                  <div @click="(e) => e.stopPropagation()">
                    <MoreOutlined />
                  </div>
                  <template #overlay>
                    <a-menu>
                      <a-menu-item @click="() => onDownload(dataRef)">Download</a-menu-item>
                      <a-menu-item @click="() => onRename(dataRef)">Rename</a-menu-item>
                      <a-menu-item @click="() => onDelete([dataRef.key])">Delete</a-menu-item>
                    </a-menu>
                  </template>
                </a-dropdown>
              </a-space>
            </a-row>
          </template>
          <template #icon="{ dataRef, expanded }">
            <template v-if="dataRef.isLeaf && dataRef.typeFile === 'tex'">
              <FileIcon class="icon-directory" :class="{ save: dataRef.isSave }" />
            </template>
            <template v-else-if="dataRef.isLeaf && dataRef.typeFile === 'img'">
              <FileImageIcon class="icon-directory" :class="{ save: dataRef.isSave }" />
            </template>
            <template v-else-if="expanded">
              <FolderOpenIcon class="icon-directory" :class="{ save: dataRef.isSave }" />
            </template>
            <template v-else>
              <FolderIcon class="icon-directory" :class="{ save: dataRef.isSave }" />
            </template>
          </template>
        </a-directory-tree>
      </div>
    </pane>
    <pane min-size="10" class="directory-div" :theme="theme" v-if="!readOnly">
      <a-row class="header-directory" align="center" justify="space-between">
        <a-space>
          <a-typography-text class="title-text-directory" strong>UNSAVED FILES</a-typography-text>
        </a-space>
      </a-row>
      <div class="tree-container">
        <a-directory-tree
          :treeData="unSaves"
          mode="inline"
          :style="{ height: '100%', borderRight: 0, overflow: 'hidden' }"
          :showIcon="false"
          @select="onChangeSelectedKeys2"
          v-model:selectedKeys="selectedKeys2"
        >
          <template #title="{ dataRef }">
            <a-row class="row-directory save" style="width: 100%; overflow: hidden; text-overflow: ellipsis;" :title="dataRef.key">
              <div style="font-weight: 600; margin-right: 8px;">{{ dataRef.title }}</div> 
              <div style="font-style: italic;">{{ dataRef.key }}</div>
            </a-row>
          </template>
        </a-directory-tree>
      </div>
    </pane>
  </Splitpanes>
  <AddNewImage
    v-model:open="openImage"
    :selectedKeys="selectedKeys"
    :listFile="initData"
    @update:files="onUpdateFiles"
  />
</template>

<script>
import { computed, defineComponent, h, inject, ref, watch, watchEffect } from 'vue'
import {
  UploadOutlined,
  DeleteFilled,
  CaretRightOutlined,
  FileAddOutlined,
  MoreOutlined
} from '@ant-design/icons-vue'
import { useRoute } from 'vue-router'
import { serviceAPI } from '@/services/API'
import { NotiError } from '@/services/notification'
import { Confirm } from '@/services/confirm'
import AddNewImage from '@/components/AddNewImage.vue'
import { notification, Button } from 'ant-design-vue'
import FileIcon from '../../public/icons/file.svg'
import FileImageIcon from '../../public/icons/file-image.svg'
import FolderIcon from '../../public/icons/folder.svg'
import FolderOpenIcon from '../../public/icons/folder-open.svg'
import { Splitpanes, Pane } from 'splitpanes'

export default defineComponent({
  components: {
    UploadOutlined,
    DeleteFilled,
    CaretRightOutlined,
    FileAddOutlined,
    FileIcon,
    FileImageIcon,
    FolderIcon,
    FolderOpenIcon,
    MoreOutlined,
    AddNewImage,
    Splitpanes,
    Pane
  },
  props: {
    initData: {
      type: Array,
      default: []
    },
    readOnly: {
      type: Boolean,
      default: false
    },
    mainFile: {
      type: Object,
      default: {}
    }
  },
  emits: ['changeSelected', 'changeSelected2', 'update:files', 'downloadFolder'],
  setup(props, { emit }) {
    const theme = inject('theme')
    const selectedKeys = ref([])
    const selectedKeys2 = ref([])
    const expandedKeys = ref([])
    const files = ref([])
    const route = useRoute()
    const popup = ref({
      typePopUp: '',
      open: ''
    })
    const paths = new Set()
    const loading = ref(false)
    const imageFile = ref()
    const isMulti = computed(() => selectedKeys.value.length > 1)
    // const open1 = ref(true)
    // const open2 = ref(true)
    const name = ref('')
    const input = ref()
    const openImage = ref(false)
    const isFirstLoad = ref(true)
    const unSaves = computed(() =>
      props.initData
        .filter((f) => f.isSave === false)
        .map((f) => ({
          key: f.path,
          title: f.name
        }))
    )

    watchEffect(() => console.log(props.initData.filter((f) => f.isSave === false)))

    watch(
      () => [props.mainFile.path],
      () => {
        if (!isFirstLoad.value) return
        selectedKeys.value = [props.mainFile.path]
        isFirstLoad.value = false
      }
    )

    const compareFile = (a, b) => {
      let aIsLeaf = a.isLeaf ? 1 : 0
      let bIsLeaf = b.isLeaf ? 1 : 0
      if (aIsLeaf !== bIsLeaf) {
        return aIsLeaf - bIsLeaf
      }

      if (a.title && b.title) {
        return a.title.localeCompare(b.title)
      } else if (a.title) {
        return -1
      } else if (b.title) {
        return 1
      }

      return 0
    }

    const onNewDirectory = (open) => {
      let parts = selectedKeys.value[0]?.split('/') || []
      if (parts.length > 0 && parts[parts.length - 1].includes('.')) parts.pop()
      parts.push('new')
      const id = parts.join('/')
      if (open === 'folder') {
        name.value = 'New folder'

        files.value = insertPath(files.value, [...parts], {
          title: '',
          key: `/${id}`,
          children: [],
          isLeaf: false,
          id: 'input',
          isSave: true
        })
        parts.pop()
        const key = parts.join('/')
        if (!!key && !expandedKeys.value.includes(key)) {
          expandedKeys.value.push(key)
          expandedKeys.value = [...expandedKeys.value]
        }
        setTimeout(() => {
          input.value.focus()
          input.value.select()
        }, 300)
      } else if (open === 'file') {
        name.value = 'New file.tex'
        files.value = insertPath(
          files.value,
          [...parts],
          {
            title: '',
            key: `/${id}`,
            isLeaf: true,
            id: 'input',
            isSave: true,
            typeFile: 'tex'
          },
          true
        )
        parts.pop()
        const key = parts.join('/')
        if (!!key && !expandedKeys.value.includes(key)) {
          expandedKeys.value.push(key)
          expandedKeys.value = [...expandedKeys.value]
        }
        setTimeout(() => {
          input.value.focus()
          input.value.setSelectionRange(0, 8)
        }, 300)
      }
    }

    const onRename = (event) => {
      const title = event.title
      const path = event.key.split('/')
      name.value = event.title
      files.value = updateNode(files.value, path, {
        title: event.id,
        key: '/' + event.key,
        id: 'input'
      })
      setTimeout(() => {
        const ext = title.split('.').pop()
        input.value.focus()
        input.value.setSelectionRange(0, title.length - ext.length - 1)
      }, 0)
    }

    const handleBlurName = (dataRef) => {
      if (dataRef.title === 'new') {
        let parts = selectedKeys.value[0]?.split('/') || []
        if (parts.length > 0 && parts[parts.length - 1].includes('.')) parts.pop()
        const oldPath = [...parts]

        const newName = generateUniqueName(name.value, parts)
        parts.push(newName)

        if (!name.value) {
          files.value = deleteNode(files.value, [...oldPath, 'new'])
          return
        }

        files.value = updateNode(files.value, [...oldPath, 'new'], {
          title: newName,
          key: parts.join('/'),
          isSave: true,
          id: 'folder'
        })
        paths.add(parts.join('/'))
        if (dataRef.typeFile === 'tex') {
          serviceAPI
            .createFile({
              versionId: route.params.versionId,
              name: newName,
              path: parts.join('/')
            })
            .then(() => {
              emit('update:files')
            })
            .catch((err) => {
              files.value = deleteNode(files.value, selectedKeys.value[0].split('/'))
              if (err.response.status == 400) {
                NotiFileList({
                  type: 'saveImage',
                  message: err.response.data.message
                })
              } else NotiError('Failed to create new file!')
            })
        }
      } else if (dataRef.title === 'folder') {
        const parts = dataRef.key.split('/')
        parts.shift()
        const updateFiles = props.initData.filter((e) => e.path.startsWith(parts.join('/')))
        parts.pop()
        const newName = generateUniqueName(name.value, parts)
        parts.push(newName)
        Promise.all(
          updateFiles.map((e) => {
            return serviceAPI
              .updateFile(e.id, {
                versionId: route.params.versionId,
                name: newName,
                path: parts.concat(e.path.split('/').slice(parts.length)).join('/'),
                shaCode: e.localShaCode || e.shaCode
              })
              .catch((err) => {
                if (err.response.status == 400) {
                  NotiFileList({
                    type: 'saveImage',
                    message: err.response.data.message,
                    id: err.response.data.id
                  })
                } else NotiError('Failed to rename this file!')
              })
          })
        ).finally(() => {
          emit('update:files')
        })
      } else {
        const parts = dataRef.key.split('/')
        parts.shift()
        parts.pop()
        const newName = generateUniqueName(name.value, parts)
        parts.push(newName)
        const file = props.initData.find((e) => e.id === dataRef.title)
        serviceAPI
          .updateFile(dataRef.title, {
            versionId: route.params.versionId,
            name: newName,
            path: parts.join('/'),
            shaCode: file.localShaCode || file.shaCode
          })
          .then(() => {
            emit('update:files')
          })
          .catch((err) => {
            files.value = deleteNode(files.value, selectedKeys.value[0].split('/'))
            if (err.response.status == 400) {
              NotiFileList({
                type: 'saveImage',
                message: err.response.data.message
              })
            } else NotiError('Failed to update this file!')
          })
      }
    }

    function generateUniqueName(baseName, parts) {
      const ext = baseName.split('.').pop()
      const noExt = ext.length === baseName.length

      const nameWithoutExt = noExt ? baseName : baseName.slice(0, baseName.length - ext.length - 1)

      let folderName = baseName
      let counter = 1
      while (paths.has([...parts, folderName].join('/'))) {
        folderName = noExt ? `${baseName} (${counter})` : `${nameWithoutExt} (${counter}).${ext}`
        counter += 1
      }
      return folderName
    }

    function insertPath(node, parts, item, isLeaf = false) {
      if (parts.length === 0) return node
      const part = parts.shift()
      let child = node.children.find((c) => c.title === part)
      if (!child) {
        child = {
          title: part,
          key: node.key ? `${node.key}/${part}` : part,
          children: [],
          isLeaf: false,
          isSave: true,
          id: item.id === 'input' ? 'input' : 'folder'
        }
        node.children.push(child)
        node.children.sort(compareFile)
        paths.add(node.key ? `${node.key}/${part}` : part)
      }
      if (parts.length === 0 && isLeaf) {
        child.isLeaf = true
        child.id = item.id
        child.typeFile = item.typeFile
        child.isSave = item.isSave
      } else {
        insertPath(child, parts, item, isLeaf)
      }
      return node
    }

    function updateNode(node, parts, newItem) {
      if (parts.length === 0) return

      const part = parts.shift()
      let child = node.children.find((c) => c.title === part)

      if (!child) return node

      if (parts.length === 0) {
        child.title = newItem.title !== undefined ? newItem.title : child.title
        child.key = newItem.key !== undefined ? newItem.key : child.key
        child.isSave = newItem.isSave !== undefined ? newItem.isSave : child.isSave
        child.id = newItem.id !== undefined ? newItem.id : child.id
        child.typeFile = newItem.typeFile !== undefined ? newItem.typeFile : child.typeFile
      } else {
        updateNode(child, parts, newItem)
      }

      return node
    }

    function deleteNode(node, parts) {
      if (parts.length === 0) return node

      const part = parts.shift()
      const childIndex = node.children.findIndex((c) => c.title === part)

      if (childIndex === -1) return node

      if (parts.length === 0) {
        node.children.splice(childIndex, 1)
      } else {
        deleteNode(node.children[childIndex], parts)
      }
      return node
    }

    function buildDirectoryStructure(files) {
      const root = { title: '', path: '', children: [] }

      files.forEach((file) => {
        const parts = file.path.split('/')
        const fileName = parts.pop()
        const leafNode = {
          title: fileName,
          key: `${root.path}/${file.path}`,
          children: [],
          isLeaf: false,
          typeFile: file.type,
          isSave: file.isSave,
          id: file.id
        }
        insertPath(root, file.path.split('/'), leafNode, true)
      })

      return root
    }

    function onFileChanged($event) {
      const target = $event.target
      if (target && target.files) {
        imageFile.value = target.files[0]
      }
    }

    const onDelete = (keys) => {
      Confirm({
        content: 'Are you sure to delete these files?',
        okText: 'Delete',
        onOk() {
          const deleteFiles = props.initData.filter(
            (e) => keys.findIndex((k) => e.path.startsWith(k)) !== -1
          )
          if (loading.value) return
          loading.value = true
          Promise.all(
            deleteFiles.map((e) =>
              serviceAPI
                .deleteFile(e.id, e.localShaCode)
                .catch((err) => {
                  if (err.response.status == 400) {
                    NotiDelete({
                      type: 'saveImage',
                      message: err.response.data.message,
                      id: err.response.data.id
                    })
                  } else NotiError('Failed to delete this file!')
                })
                .finally(() => {
                  loading.value = false
                })
            )
          ).finally(() => {
            emit('update:files')
            selectedKeys.value = []
          })
        }
      })
    }

    const NotiFileList = ({ type = '', filePath = '', message }) => {
      notification.error({
        message: message || 'Files list has been changed',
        description: 'Please update files list before create new file!',
        btn: () =>
          h(
            Button,
            {
              type: 'primary',
              onClick: () => {
                notification.close(`${type}${filePath}`)
                emit('update:files')
              }
            },
            { default: () => 'Update' }
          ),
        key: `${type}${filePath}`
      })
    }

    const NotiDelete = ({ type = '', filePath = '', message, id, shaCode }) => {
      notification.error({
        message: message || 'Files list has been changed',
        description: 'Do you want to delete?',
        btn: () =>
          h(
            Button,
            {
              type: 'text',
              onClick: () => {
                notification.close(`${type}${filePath}`)
                serviceAPI.deleteFile(id, shaCode).catch((err) => null)
              }
            },
            { default: () => 'Delete' }
          ),
        key: `${type}${filePath}`
      })
    }

    const onChangeSelectedKeys = (event) => {
      selectedKeys2.value = []
      // không chọn file nào
      if (event.length === 0 || !event) {
        emit('changeSelected', null)
        return
      }
      // chọn 1 file tex hoặc ảnh
      if (event.length === 1 && event[0].includes('.')) {
        emit(
          'changeSelected',
          props.initData.find((e) => e.path === event[0])
        )
        return
      }
      // chọn nhiều file hoặc chọn folder
      emit('changeSelected', null)
    }

    const onChangeSelectedKeys2 = (event) => {
      selectedKeys.value = []
      emit(
        'changeSelected2',
        props.initData.find((e) => e.path === event[0])
      )
    }

    watchEffect(() => {
      files.value = buildDirectoryStructure(props.initData)
    })

    const onDownload = (dataRef) => {
      if (!dataRef.isLeaf) {
        emit('downloadFolder', dataRef)
        return
      }
      if (dataRef.typeFile === 'tex') {
        const fileInfo = props.initData.find((e) => e.id === dataRef.id)
        let file = new Blob([localStorage.getItem(dataRef.id) || fileInfo.content], {
          type: 'text/plain'
        })
        const link = document.createElement('a')
        link.href = window.URL.createObjectURL(file)
        link.setAttribute('download', dataRef.title)
        document.body.appendChild(link)
        link.click()
        document.body.removeChild(link)
        return
      }

      if (loading.value) return

      loading.value = true
      serviceAPI
        .downloadFile(dataRef.id)
        .then((res) => {
          const url = window.URL.createObjectURL(new Blob([res.data]))
          const link = document.createElement('a')
          link.href = url
          link.setAttribute('download', dataRef.title)
          document.body.appendChild(link)
          link.click()
          document.body.removeChild(link)
        })
        .catch((err) => {
          NotiError('Failed to download this file!')
        })
        .finally(() => {
          loading.value = false
        })
    }

    const onUpdateFiles = () => {
      emit('update:files')
    }

    return {
      selectedKeys,
      selectedKeys2,
      expandedKeys,
      files,
      onNewDirectory,
      imageFile,
      onFileChanged,
      isMulti,
      onDelete,
      onChangeSelectedKeys,
      // open1,
      // open2,
      onDownload,
      popup,
      name,
      handleBlurName,
      input,
      onRename,
      openImage,
      onUpdateFiles,
      theme,
      unSaves,
      onChangeSelectedKeys2
    }
  }
})
</script>

<style lang="scss">
@import '@/variable.scss';

@mixin apply-theme($theme) {
  $color-border: map-get($theme, color-border);
  $color-background: map-get($theme, color-background);
  $text-secondary: map-get($theme, text-secondary);
  $text-primary: map-get($theme, text-primary);
  $text-modify: map-get($theme, text-modify);
  $color-background-dir: map-get($theme, color-background-dir);

  background-color: $color-background-dir;

  .toggle-ic,
  .title-text-directory {
    color: $text-secondary;
  }

  .ant-tree-switcher {
    color: $text-primary !important;
  }

  .icon-directory {
    path {
      stroke: $text-modify;
    }
    &.save path {
      stroke: $text-primary;
    }
  }

  .row-directory {
    color: $text-modify;
    &.save {
      color: $text-primary;
    }
  }
}

.icon-select {
  width: 20px;
}

.icon-btn-dir {
  padding: 2px;
  height: 24px;
}

.ant-tree-treenode-selected .row-directory .anticon {
  opacity: 100;
}

.directory-div {
  &[theme='light'] {
    @include apply-theme($theme-light);
  }
  &[theme='dark'] {
    @include apply-theme($theme-dark);
  }

  .header-directory {
    padding: 4px 8px;
    background: rgba(109, 91, 208, 0.09);
    cursor: pointer;

    .control-btns {
      transition: all 0.1s ease-in-out;
    }

    .title-text-directory {
      font-size: 12px;
      line-height: 30px;
    }
  }
  .icon-directory {
    transform: scale(0.8);
    position: relative;
    top: 2px;
  }

  .ant-tree {
    border-radius: 0;

    .ant-tree-treenode {
      padding: 4px 0 8px 0;
      font-size: 18px;
    }

    .ant-tree-treenode:hover {
      background: rgba(109, 91, 208, 0.05);
    }

    .ant-tree-node-content-wrapper {
      width: calc(100% - 24px);
    }
  }

  .tree-container {
    max-height: 0;
    overflow: hidden;
    transition: max-height 0.3s ease-in-out;

    .row-directory {
      display: inline-flex;
      flex-wrap: nowrap;
      width: calc(100% - 30px);

      > span {
        text-overflow: ellipsis;
        white-space: nowrap;
        width: auto;
        overflow: hidden;
      }

      .anticon {
        opacity: 0;
        transition: all 150ms ease-in-out;
      }

      &:hover .anticon {
        opacity: 100;
      }
    }
  }
  .control-btns {
    opacity: 100;
  }
  .tree-container {
    max-height: calc(100% - 38px);
    overflow: auto;
  }
}
</style>
