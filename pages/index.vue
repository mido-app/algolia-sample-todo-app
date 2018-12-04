<template>
  <div>
    <b-button @click="openModal">追加</b-button>
    <b-modal v-model="modalIsVisible"
             @ok="addTodo"
             @calcel="clearInput">
      <b-form-group label="Title" label-for="title">
       <b-form-input id="title" type="text" v-model="todoInput.title"></b-form-input>
      </b-form-group>
      <b-form-group label="Description" label-for="description">
       <b-form-input id="description" type="text" v-model="todoInput.description"></b-form-input>
      </b-form-group>
    </b-modal>
  </div>
</template>

<script>
import * as algoliasearch from 'algoliasearch'
import config from '~/algolia.config.js'

const client = algoliasearch(config.appId, config.apiKey)
const index = client.initIndex('todo')

export default {
  data () {
    return {
      todoInput: {
        title: '',
        description: ''
      },
      modalIsVisible: false
    }
  },
  methods: {
    clearInput () {
      this.todoInput.title = ''
      this.todoInput.description = ''
    },
    openModal () {
      this.modalIsVisible = true
    },
    async addTodo () {
      let content = await index.addObject(this.todoInput)
      this.clearInput()
    }
  } 
}
</script>
