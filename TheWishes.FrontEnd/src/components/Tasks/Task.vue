<template>
  <li class="my-3">
    <on-click-outside :do="handleClickOutside">
      <form @submit.prevent="updateTask">
        <form-error :error="error"></form-error>

        <!-- Task -->
        <div class="bg-white leading-none rounded-lg shadow overflow-hidden p-2 mb-3">
          <!-- Update form -->
          <div v-if="editTask" class="flex flex-col">
            <input v-focus @keyup.esc="cancelEdit" v-model="form.title" placeholder="What needs to be done?" class="w-full mb-2 pb-2 px-2 focus:outline-none text-lg font-semibold border-b" type="text">

            <div class="flex items-center text-xs">
              <fa :icon="['far', 'clock']" class="mr-1 text-grey-dark" />
              <datetime type="datetime" v-model="form.due_at" placeholder="Due at" :minute-step="5" input-class="text-grey-dark"></datetime>

              <span v-if="form.due_at" @click="clearDueAt" class="flex-none rounded-full bg-grey hover:bg-red h-6 w-6 cursor-pointer flex items-center justify-center shadow">
                <fa icon="times" class="text-white" />
              </span>
            </div>
          </div>

          <div v-else class="d-flex align-items-center">
            <div class="flex-grow-1">
              <p @click="editTask = true" class="font-semibold text-lg mx-1 my-0 text-left flex-auto cursor-pointer" :class="{'line-through text-grey' : task.is_completed}">{{ task.title }}</p>

              <span v-if="task.due_at" @click="editTask = true" :title="toDate(task)" class="flex flex-no-shrink mr-2 mt-2 px-2 py-1 text-xs cursor-pointer" :class="[task.is_completed ? 'line-through text-grey' : 'text-grey-dark']">
                <fa :icon="['far', 'clock']" class="mr-1" /> {{ fromNow(task) }}
              </span>
            </div>

            <!-- Checkbox -->
            <div @click="toggleCompleted" :class="[task.is_completed ? 'bg-success' : 'border-2  bg-white', {'cursor-not-allowed' : isToggleLoading}]" class="rounded-circle border h-6 w-6 cursor-pointer d-flex align-items-center justify-content-center">
              <fa v-if="isToggleLoading" icon="spinner" :class="[task.is_completed ? 'text-white' : 'text-success']" spin />
              <fa v-else icon="check" class="text-white" :class="{'hover:text-success' : ! task.is_completed}" />
            </div>
          </div>
        </div>

        <!-- Update buttons -->
        <div v-if="editTask" class="flex items-center justify-between mt-2">
          <div class="flex items-center">
            <loading-button
              @click.native="updateTask"
              :isLoading="isUpdateLoading"
              :class="{'opacity-50 cursor-not-allowed' : isDisabled}"
              icon="check"
              class="btn-indigo text-sm">
                Save
            </loading-button>

            <span @click="cancelEdit" class="ml-4 text-grey-darker text-sm cursor-pointer hover:underline">Cancel</span>
          </div>

          <loading-button
            @click.native="removeTask"
            :isLoading="isRemoveLoading"
            :class="[isRemoveLoading ? 'opacity-50 cursor-not-allowed' : 'cursor-pointer hover:underline hover:text-red']"
            type="button"
            icon="trash"
            class="mx-4 text-grey-darker text-sm">
              Delete
          </loading-button>
        </div>
      </form>
    </on-click-outside>
  </li>
</template>

<script>
import moment from 'moment'
import Form from '@/utils/Form'
import OnClickOutside from '@/components/OnClickOutside'
import 'vue-datetime/dist/vue-datetime.css'

export default {
  components: { OnClickOutside },
  props: {
    task: {
      type: Object,
      required: true
    }
  },

  data () {
    return {
      isToggleLoading: false,
      isRemoveLoading: false,
      isUpdateLoading: false,
      editTask: false,
      error: null,
      form: new Form({
        title: this.task.title,
        due_at: this.task.due_at
      })
    }
  },

  computed: {
    isDisabled () {
      return this.form.title === '' || this.isUpdateLoading
    },

    isNotLoading () {
      return !(this.isToggleLoading || this.isRemoveLoading || this.isUpdateLoading)
    }
  },

  methods: {
    fromNow (task) {
      return moment(task.due_at).fromNow()
    },

    toDate (task) {
      return moment(task.due_at).format('dddd, MMMM Do YYYY, h:mm:ss a')
    },

    toggleCompleted () {
      if (this.isToggleLoading) {
        return false
      }

      this.isToggleLoading = true
      this.error = null

      this.$store.dispatch('updateTask', {
        task: this.task,
        form: { is_completed: !this.task.is_completed }
      })
        .then(() => {
          this.isToggleLoading = false
        })
        .catch(error => {
          this.error = error.response.data
          this.isToggleLoading = false
        })
    },

    updateTask () {
      if (this.isDisabled) {
        return false
      }

      this.isUpdateLoading = true
      this.error = null

      this.$store.dispatch('updateTask', {
        task: this.task,
        form: {
          title: this.form.title,
          due_at: this.form.due_at ? moment(this.form.due_at).format('YYYY-MM-DD HH:mm:ss') : null
        }
      })
        .then(data => {
          this.form.due_at = data.due_at

          this.isUpdateLoading = false
          this.editTask = false
        })
        .catch(error => {
          this.isUpdateLoading = false
          this.error = error.response.data
        })
    },

    cancelEdit () {
      this.editTask = false
      this.error = null

      this.form.title = this.task.title
      this.form.due_at = this.task.due_at
    },

    clearDueAt () {
      this.form.due_at = null
    },

    removeTask () {
      if (this.isRemoveLoading || !window.confirm('Are you sure ? Your task will be deleted forever.')) {
        return false
      }

      this.isRemoveLoading = true
      this.error = null

      this.$store.dispatch('removeTask', this.task)
        .catch(error => {
          this.error = error.response.data
        })
    },

    handleClickOutside () {
      this.error = null

      if (this.editTask && this.isNotLoading) {
        this.cancelEdit()
      }
    }
  }
}
</script>
