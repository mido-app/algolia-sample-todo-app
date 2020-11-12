<template>
  <div class="p-2">
    <b-form-row>
      <b-col>
        <b-form-input type="text" v-model="query" />
      </b-col>
      <b-col cols="2">
        <b-button block @click="searchTodo">検索</b-button>
      </b-col>
    </b-form-row>
    <b-button class="my-2" variant="primary"  block @click="openRegisterModal">TODOを追加</b-button>

    <!-- TODO一覧 -->
    <b-card class="my-2"
            v-for="todo in highlightedTodoList" 
            :key="todo.objectID">
      <b-card-title v-html="todo.title"></b-card-title>
      <p v-html="todo.description"></p>
      <b-button v-if="!todo.done" variant="primary" @click="makeTodoDone(todo.objectID)">完了</b-button>
      <b-button v-else disabled>完了済</b-button>
      <b-button @click="openUpdateModal(todo.objectID)">編集</b-button>
      <b-button @click="deleteTodo(todo.objectID)">削除</b-button>
    </b-card>

    <!-- 登録用ダイアログ -->
    <b-modal v-model="registerModalIsVisible"
             @ok="registerTodo"
             @calcel="clearInput">
      <b-form-group label="Title" label-for="title">
       <b-form-input id="register-title" type="text" v-model="todoInput.title" />
      </b-form-group>
      <b-form-group label="Description" label-for="description">
       <b-form-input id="register-description" type="text" v-model="todoInput.description" />
      </b-form-group>
    </b-modal>

    <!-- 更新用ダイアログ -->
    <b-modal v-model="updateModalIsVisible"
             @ok="updateTodo"
             @calcel="clearInput">
      <b-form-group label="Title" label-for="title">
       <b-form-input id="update-title" type="text" v-model="todoInput.title" />
      </b-form-group>
      <b-form-group label="Description" label-for="description">
       <b-form-input id="update-description" type="text" v-model="todoInput.description" />
      </b-form-group>
    </b-modal>
  </div>
</template>

<script>
import * as algoliasearch from 'algoliasearch'
import { createNullCache } from '@algolia/cache-common'
import config from '~/algolia.config.js'

const client = algoliasearch(config.appId, config.apiKey, {
  responsesCache: createNullCache()
})
const index = client.initIndex('todo')

index.setSettings({
  attributesToHighlight: [
    'title',
    'description'
  ],
  highlightPreTag: '<em class="search-highlight">',
  highlightPostTag: '</em>'
})

export default {
  async asyncData () {
    let searchResult = await index.search('')
    return {
      todoList: searchResult.hits
    }
  },
  data () {
    return {
      query: '',
      todoList: [],
      todoInput: {
        title: '',
        description: '',
        done: false
      },
      registerModalIsVisible: false,
      updateModalIsVisible: false,
      updateTargetObjectID: null
    }
  },
  methods: {
    clearInput () {
      this.todoInput.title = ''
      this.todoInput.description = '',
      this.todoInput.done = false
    },
    openRegisterModal () {
      this.registerModalIsVisible = true
    },
    async registerTodo () {
      let todo = Object.assign({}, this.todoInput)
      let content = await index.saveObject(todo, { autoGenerateObjectIDIfNotExist: true })
      await this.searchTodo()
      this.clearInput()
    },
    openUpdateModal (objectID) {
      const todo = this.getTodoByObjectID(objectID)
      console.log(todo)
      this.todoInput.title = todo.title
      this.todoInput.description = todo.description
      this.updateTargetObjectID = objectID
      this.updateModalIsVisible = true
    },
    async updateTodo () {      
      await index.saveObject({
        objectID: this.updateTargetObjectID,
        ...this.todoInput
      })
      await this.searchTodo()
      this.clearInput()
    },
    async deleteTodo (objectID) {
      await index.deleteObject(objectID)
      await this.searchTodo()
    },
    async makeTodoDone (objectID) {
      await index.partialUpdateObject({
        objectID: objectID,
        done: true
      })
      await this.searchTodo()
    },
    async searchTodo () {
      let searchResult = await index.search(this.query)
      this.todoList = searchResult.hits
    },
    getTodoByObjectID(objectID) {
      return this.todoList.filter(todo => todo.objectID === objectID)[0]
    }
  },
  computed: {
    highlightedTodoList () {
      return this.todoList.map(todo => {
        let highlightedTodo = Object.assign({}, todo)
        if (todo._highlightResult && todo._highlightResult.title.matchLevel !== 'none') {
          highlightedTodo.title = todo._highlightResult.title.value
        }
        if (todo._highlightResult && todo._highlightResult.description.matchLevel !== 'none') {
          highlightedTodo.description = todo._highlightResult.description.value
        }
        return highlightedTodo
      })
    }
  }
}
</script>

<style>
.search-highlight {
  background-color: yellow;
}
</style>
