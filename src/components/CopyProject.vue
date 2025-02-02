<template>
  <a-modal title="Copy Project" :open="open" :footer="null" @cancel="onCancel">
    <a-form :model="formState" name="basic" autocomplete="off" @finish="onFinish" layout="vertical">
      <a-form-item
        label="Project Name"
        name="name"
        :rules="[{ required: true, message: 'Project name is required!' }]"
      >
        <a-input v-model:value="formState.name" />
      </a-form-item>

      <a-form-item
        label="Version"
        name="version"
        :rules="[{ required: true, message: 'Version is required!' }]"
      >
        <a-select v-model:value="formState.version" :options="versionOptions"></a-select>
      </a-form-item>

      <a-form-item>
        <a-row justify="end">
          <a-space>
            <a-button @click="onCancel">Cancel</a-button>
            <a-button type="primary" html-type="submit">Submit</a-button>
          </a-space>
        </a-row>
      </a-form-item>
    </a-form>
  </a-modal>
</template>
<script>
import { serviceAPI } from '@/services/API'
import { NotiError, NotiSuccess } from '@/services/notification'
import { computed, defineComponent, reactive, watch } from 'vue'

export default defineComponent({
  props: {
    open: {
      type: Boolean,
      default: false
    },
    initData: {
      type: Object,
      default: {
        name: '',
        version: ''
      }
    },
    versions: {
      type: Array,
      default: []
    }
  },
  emits: ['update:open', 'success'],
  setup(props, { emit }) {
    const formState = reactive({
      name: props.initData.name + ' (Copy)',
      version: props.initData.version
    })

    watch(
      () => [props.initData],
      () => {
        Object.assign(formState, {
          name: props.initData.name + ' (Copy)',
          version: props.initData.version
        })
      }
    )
    const versionOptions = computed(() =>
      props.versions.map((e) => ({ label: e.description, value: e.id }))
    )

    const onFinish = (values) => {
      serviceAPI
        .copyProject({
          name: values.name,
          versionId: values.version
        })
        .then(() => {
          NotiSuccess('Success')
          emit('update:open', false)
          emit('success')
        })
        .catch(() => NotiError('Failed to copy this project!'))
    }

    const onCancel = () => {
      emit('update:open', false)
    }

    return {
      versionOptions,
      formState,
      onFinish,
      onCancel
    }
  }
})
</script>
